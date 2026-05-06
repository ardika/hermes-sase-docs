---
layout: default
title: 7. Keamanan & Threat Model
nav_order: 8
permalink: /docs/keamanan/
---

# 7. Keamanan & Threat Model
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 7.1 Prinsip dasar

1. **PrivateKey WireGuard ada di Supabase** — di kolom `user_data.configuration`, di-populate oleh admin tooling. Client baca + passthrough ke Helper. **Konsekuensi threat model penting**: kalau Supabase compromise, semua PrivateKey user bocor. Lihat §7.4.
2. **Helper Service adalah trust boundary lokal** — dia satu-satunya komponen yang punya privilege admin/root, dengan API minimal & well-defined.
3. **Supabase RLS adalah trust boundary backend** — user hanya bisa lihat/edit row sendiri (`uid() = uid` di policy "Allow all usage").
4. **Client tidak `INSERT` / `UPDATE` ke `user_data`** — disiplin di kode (RLS allow, tapi tidak kita pakai). Schema dilarang diubah oleh implementasi ini.
5. **PSK production tidak dipakai** — sample data 5 user menunjukkan `PresharedKey` tidak ada di `configuration`. Jangan asumsikan ada.

## 7.2 Trust boundaries

```
┌─────────────────────────────────────────────────────────────────┐
│  Client device (Windows / Mac)                                  │
│                                                                 │
│  ┌─────────────────────────┐     named-pipe / unix-socket      │
│  │  Hermes UI (user mode)  │ ←─── (validated, authenticated) ──┐│
│  │  - User JWT             │                                   ││
│  │  - WG private key       │                                   ││
│  │   (DPAPI/Keychain)      │                                   ││
│  └────────────┬────────────┘                                   ││
│               │ HTTPS + JWT user                               ││
│               ▼                                                ││
│  ┌─────────────────────────┐                                   ││
│  │ Supabase user_data      │  <── trust boundary 1 (RLS)       ││
│  │ (read-only client side) │                                   ││
│  └─────────────────────────┘                                   ││
│                                                                ││
│  ┌─────────────────────────────────────────────────────────────┘│
│  │  Hermes Helper Service (LocalSystem / root)                   │
│  │  - Stateless                                                  │
│  │  - Verb whitelist                                             │
│  │  - Caller authentication                                      │
│  │  - No network access                                          │
│  └────────────┬──────────────────────────────────────────────────┘
│               │
│               ▼
│  ┌─────────────────────────┐
│  │ tunnel.dll embedded (Win) /  │  <── OS service / kernel
│  │ wireguard-go bundled (Mac)   │
│  └─────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

## 7.3 Threat model

| Threat | Defended? | Mitigasi |
|---|---|---|
| Reverse-engineer UI binary → ekstrak credential | ✅ | Tidak ada API key privileged di binary; hanya anon key Supabase yang dilindungi RLS |
| Steal `.conf` file di disk endpoint | ⚠️ Partial | Win: DPAPI-encrypted oleh WireGuard tunnel service; Mac: mode 0600 root-owned. Karena tidak ada PSK, attacker yang dapat `.conf` punya akses penuh ke gateway. |
| Attack lewat IPC (user lain di multi-user system) | ✅ | Helper authenticate caller via Win token / Unix peer creds (lihat §7.5) |
| User compromise → run binary jahat sebagai dia | ⚠️ | Helper accept dari user yang sama; bisa request operasi WireGuard. Mitigasi: signed binary check (lihat §7.6) |
| **Compromise Supabase host** | ❌ Critical | **Semua PrivateKey user bocor** (ada di `user_data.configuration`). Mitigasi: hardening Supabase host, RLS, audit log, rotate API keys. Lihat §7.4. |
| Compromise Supabase admin (service_role) | ❌ | Out of scope — backend compromise = total game over. Audit + key rotation. |
| Compromise gateway WireGuard | ❌ | Threat di-luar scope. Mitigasi: rotate semua peer keys via admin tooling. |
| MITM antara UI ↔ Supabase | ✅ | HTTPS + (opsional) cert pinning |
| MITM antara UI ↔ Helper (named pipe / unix socket) | ✅ | Lokal kernel — tidak melewati network. Authenticate caller. |
| MITM antara client ↔ gateway WG | ✅ | WireGuard ChaCha20-Poly1305 + Curve25519 (PSK tidak dipakai di production) |
| Replay attack pada WireGuard handshake | ✅ | Built-in: nonce + counter |
| Steal user JWT dari memory | ⚠️ Partial | TTL pendek, refresh rotated, simpan refresh di OS keychain |
| DNS leak | ✅ | DNS dari `configuration` field, otomatis di-set oleh `wg-quick` / WireGuard tunnel |
| IP leak saat handshake bermasalah | ⚠️ | WireGuard config production tidak include kill switch; kalau handshake gagal, trafik fallback ke route default. Kalau perlu kill switch, koordinasi dengan admin tooling untuk tambah `PostUp` rule di config. |

## 7.4 Aliran kunci & data sensitif

| Asset | Disimpan di mana | Kapan ada di memori | Yang boleh akses |
|---|---|---|---|
| WG PrivateKey | **Supabase `user_data.configuration` (text)** | Memori UI app saat passthrough ke Helper, lalu di disk WireGuard config | Admin (yang manage `configuration`), UI temp, Helper file |
| WG PublicKey gateway | `[Peer]` block di `configuration` | Saat passthrough | (publik) |
| User JWT | OS keychain encrypted | Saat HTTP request | UI app |
| Supabase service_role key | **TIDAK ADA di client** | N/A | Hanya backend / admin |
| Helper IPC pipe | Kernel-managed | N/A | Process yang punya akses |

> **Konsekuensi disain "PrivateKey di Supabase":** ini memang sub-optimal dari perspective threat model (compromise Supabase = total user PrivateKey leak). Mitigasi yang relevan dilakukan di sisi ops, bukan di kode client:
>
> - Supabase host hardening (TLS, host firewall, OS update, separate DB user)
> - Backup encrypted at rest
> - Audit access ke `user_data.configuration` lewat Supabase audit log
> - Rotate kunci di gateway secara periodik (admin tooling generate config baru, replace `configuration`)
>
> Implementasi client tidak punya leverage untuk fix masalah ini. Dokumen ini menerima konsekuensi dan fokus pada eksekusi yang benar di client side.

## 7.5 Caller authentication ke Helper

### 7.5.1 Windows

`NamedPipeServerStream` punya `RunAsClient` yang impersonate caller. Bisa ambil `WindowsIdentity` dan check:

- Apakah user yang sama dengan yang menjalankan `HermesNetwork360Guard.exe`?
- Atau group membership tertentu?

```csharp
public sealed class WindowsCallerAuthenticator
{
    public bool Authenticate(NamedPipeServerStream pipe)
    {
        WindowsIdentity? callerId = null;
        pipe.RunAsClient(() =>
        {
            callerId = WindowsIdentity.GetCurrent();
        });
        if (callerId is null) return false;

        // Policy: hanya terima dari user yang sedang interactive di session yang sama
        // Untuk simple: trust kalau caller adalah Authenticated User
        return callerId.IsAuthenticated;
    }
}
```

Lebih ketat: simpan PID + username UI app saat install, di Helper check apakah caller punya identity sama.

### 7.5.2 macOS / Linux

Pakai `getpeereid()` (macOS) atau `SO_PEERCRED` (Linux) untuk dapat UID + GID peer:

```csharp
[DllImport("libc")]
private static extern int getpeereid(int sockfd, out uint euid, out uint egid);

public bool AuthenticateMacPeer(Socket sock)
{
    var fd = (int)sock.Handle;
    if (getpeereid(fd, out var euid, out var egid) != 0)
        return false;

    // Policy: hanya terima dari user yang punya session GUI aktif (atau dari list approved UIDs)
    return euid >= 500;   // skip system accounts
}
```

Permission socket file `/var/run/hermes-helper.sock` di-set 0666 (semua user bisa connect) atau 0660 dengan group khusus, tergantung policy.

### 7.5.3 Pencegahan replay

JSON-RPC frame **tidak** signed atau timestamped saat ini. Karena IPC tidak melewati network, replay attack hanya relevan kalau attacker sudah dapat akses ke pipe/socket lokal — yang berarti dia sudah punya privilege user. Tambahan signing tidak menambah security signifikan.

## 7.6 Helper integrity

### 7.6.1 Signed binary

Helper dan UI **harus signed dengan signing identity yang sama**:
- Windows: Authenticode signed dengan EV / OV cert
- macOS: Developer ID Application

Pada install, OS verify signature. Saat runtime, Helper bisa **verify caller signature** untuk meningkatkan kepercayaan:

```csharp
// Windows: cek signature dari process binary caller
public bool VerifyCallerSignature(int callerPid)
{
    var process = Process.GetProcessById(callerPid);
    var binPath = process.MainModule?.FileName;
    if (binPath is null) return false;

    // Pakai WinVerifyTrust API atau cek subject di Authenticode
    return AuthenticodeVerifier.IsSignedBy(binPath, "Hermes Network Inc.");
}
```

### 7.6.2 Update protection

Helper binary dilindungi oleh ACL Windows Service file (default: hanya admin yang bisa overwrite). Updater Hermes harus jalan sebagai admin saat replace binary — lewat trusted updater service / package manager (e.g. MSIX update).

## 7.7 Kill switch

Untuk role yang sensitive (executive / contractor), tunnel **harus** up; kalau gagal, semua trafik di-block (no leak).

WireGuard config dengan kill switch (Linux/Mac):

```ini
[Interface]
PrivateKey = ...
Address = 10.99.0.42/32
PostUp   = iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown  = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
```

Windows pakai `BlockUntunneledTraffic` adapter setting atau Windows Firewall rule:

```powershell
# Helper register firewall rule saat tunnel UP
New-NetFirewallRule -DisplayName "Hermes-SASE-KillSwitch" `
  -Direction Outbound -InterfaceAlias "Hermes" -Action Allow
New-NetFirewallRule -DisplayName "Hermes-SASE-KillSwitch-Block" `
  -Direction Outbound -Action Block -Profile Any
```

Kill switch dapat di-implementasi dengan dua cara (admin tooling pilih sesuai policy):

- **Inline di `configuration`** — admin tambah `PostUp` / `PreDown` line ke INI yang di-populate ke Supabase. Client passthrough ke Helper, Helper tulis ke disk, WireGuard tunnel service jalankan saat up/down.
- **Out-of-band via Helper verb baru** — kalau perlu logic kompleks yang tidak fit di INI, tambah verb `EnableKillSwitch(name)` di Helper. Tetap whitelist input.

Karena schema Supabase tidak boleh diubah, kill switch flag tidak bisa disimpan sebagai kolom terpisah; harus encode di dalam `configuration` (kalau pilihan pertama).

## 7.8 DNS leak prevention

Set `DNS = ...` di interface block. Untuk Windows, ini install resolver di adapter tunnel. Untuk Mac, `wg-quick` jalankan `resolvconf` setup.

Verifikasi tidak leak setelah connect:

```bash
# Win
Get-DnsClientServerAddress

# Mac
scutil --dns | head -20

# Test
nslookup example.com
# Server harus DNS dari config (mis. 10.0.0.1), bukan ISP
```

## 7.9 Storage credentials

| Asset | Windows | macOS |
|---|---|---|
| WG keypair lokal | ❌ TIDAK ADA — admin yang manage di `user_data.configuration`, client tidak generate keypair |
| Supabase refresh token | `%LOCALAPPDATA%\HermesNetwork360Guard\session.bin` (DPAPI) | macOS Keychain via `/usr/bin/security` |
| Helper config | N/A (stateless) | N/A |
| WG `.conf` (active) | `C:\Program Files\WireGuard\Data\Configurations\<name>.conf.dpapi.aes` | `/etc/wireguard/<name>.conf` (root:wheel, 0600) |
| Cache config terakhir (UI memory) | Disposed saat app close | Sama |

## 7.10 Audit logging

Karena schema Supabase tidak boleh diubah, **tidak ada tabel `sase_audit_log` baru**. Audit hanya dilakukan di sisi client (app log) dan OS log.

### 7.10.1 Yang harus di-log

| Event | Lokasi | Retention |
|---|---|---|
| Helper RPC request | Event Viewer (Win) / `os_log` (Mac) | sesuai OS log default |
| Connect / Disconnect / Reconnect | Hermes app log lokal (`%LOCALAPPDATA%\HermesNetwork360Guard\app.log`) | 7–30 hari (sesuai rotation policy existing) |
| Config refresh + hash compare hasil | App log lokal | 7 hari |
| Auth failure di Helper | Event Viewer warning | sesuai OS default |

### 7.10.2 Yang JANGAN di-log

- WG PrivateKey
- Raw isi `configuration` (mengandung PrivateKey)
- User JWT raw value
- Refresh token
- Supabase service_role / admin tokens

Pattern aman:

```csharp
_log.LogInformation("Helper RPC: {Method} duration_ms={Duration}", method, duration);
_log.LogInformation("Config refresh: hash_changed={Changed} length={Len}", changed, length);

// JANGAN:
// _log.LogInformation("Config: {Config}", configIni);   ❌  mengandung PrivateKey
```

> Kalau di kemudian hari diperlukan audit terpusat (mis. SOC 2), opsi:
> - **Reuse tabel existing** — tambah field di Supabase **dari sisi admin tooling**, bukan dari implementasi client. Client tetap tidak `INSERT`/`UPDATE` ke Supabase.
> - **External logging service** — push app log ke Graylog (sudah ada di Hermes — `log.hermesnetwork.cloud`).
> - **Hermes Network 360 Guard Log Report Browser** — pakai infrastructure yang sudah ada (lihat `LogReportService` repo) untuk upload encrypted log periodic.

## 7.11 Key rotation

### 7.11.1 WG keypair per user

**Client tidak rotate keypair.** Admin tooling yang manage:

1. Admin generate keypair baru
2. Admin update `user_data.configuration` dengan PrivateKey + PublicKey baru, plus register peer baru di gateway
3. Client polling 5 menit detect hash change → auto apply
4. Tunnel re-handshake dengan keypair baru, tanpa user interaksi

Tidak ada code path "user rotate key" di client. Kalau user mencurigai compromise, eskalasi ke admin / ops.

### 7.11.2 PSK

Production tidak pakai PSK. Kalau ke depan admin tooling tambah PSK di `configuration`, client otomatis support (passthrough INI ke Helper, WireGuard handle PSK natively).

### 7.11.3 Gateway rotation

Admin update gateway endpoint atau public key → update `configuration` semua user → client polling detect change dalam 5 menit → apply.

### 7.11.4 Helper signing certificate

Apple Developer ID + Windows Authenticode cert: rotate sebelum expire. Re-sign + re-deploy installer. Tunnel tetap jalan dengan binary lama sampai user update.

## 7.12 Compliance considerations

| Standar | Yang relevan dari arsitektur ini |
|---|---|
| **SOC 2** | RLS enforcement, key rotation oleh admin tooling, helper signed binary, app log retention |
| **ISO 27001** | Threat model di sini (terutama §7.4 PrivateKey-di-Supabase), incident response, signed binary |
| **HIPAA** | BAA dengan Supabase host, encryption at rest di DB level (Postgres + filesystem), audit log via existing Hermes infra |
| **GDPR** | Data minimization (logging policy §7.10.2), right-to-erasure (admin delete user → cascade ke `user_data` row) |

## 7.13 Disaster recovery

| Skenario | Action |
|---|---|
| Helper Service crash / unresponsive | UI tampilkan banner "Helper not running, please reinstall". Re-register service via installer. |
| Supabase tidak reachable | Cached config terakhir di-pakai (kalau ada). Tunnel tetap up. |
| Gateway WG down | Health monitor detect, status "Faulted". User notif. |
| Compromise gateway | Admin tooling: rotate gateway keypair + update semua `user_data.configuration` dengan endpoint+pubkey baru. Client auto-refresh dalam 5 menit. |
| User device hilang/dicuri | Admin tooling: clear / null-kan `user_data.configuration` user yang hilang + revoke peer di gateway. Client tunnel drop saat handshake gagal. |
| Compromise Supabase host | **Catastrophic**: semua PrivateKey bocor. Rotate gateway key + regenerate semua user `configuration`. Audit & investigation. |
| Compromise Supabase admin (service_role) | Rotate service_role key. Audit semua change ke `user_data`. |

---

[← Bab 6 Connection Flow]({{ site.baseurl }}{% link docs/06-connection-flow.md %}){: .btn }
[Bab 8 — Dukungan macOS →]({{ site.baseurl }}{% link docs/08-mac-support.md %}){: .btn .btn-primary }
