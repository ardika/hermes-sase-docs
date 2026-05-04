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

1. **Private key WireGuard tidak pernah meninggalkan client device** — disimpan di DPAPI/Keychain, di-inject oleh client saat apply config.
2. **Helper Service adalah trust boundary lokal** — dia satu-satunya komponen yang punya privilege admin/root, dengan API minimal & well-defined.
3. **Supabase RLS adalah trust boundary backend** — user hanya bisa lihat/edit row sendiri.
4. **`wg_config` di Supabase di-populate admin (server_role)** — client tidak boleh ubah field ini.
5. **PSK per-peer** disertakan dalam config untuk defense in depth.

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
│  │ wireguard.exe / wg-quick │  <── OS service / kernel
│  └─────────────────────────┘
└─────────────────────────────────────────────────────────────────┘
```

## 7.3 Threat model

| Threat | Defended? | Mitigasi |
|---|---|---|
| Reverse-engineer UI binary → ekstrak credential | ✅ | Tidak ada API key privileged di binary; hanya anon key Supabase yang dilindungi RLS |
| Steal `.conf` file di disk | ⚠️ Partial | Win: DPAPI encrypt; Mac: mode 0600 root-owned. Attacker juga butuh PSK kalau pakai PSK. |
| Attack lewat IPC (user lain di multi-user system) | ✅ | Helper authenticate caller via Win token / Unix peer creds (lihat §7.5) |
| User compromise → run binary jahat sebagai dia | ⚠️ | Helper accept dari user yang sama; bisa request operasi WireGuard. Mitigasi: signed binary check (lihat §7.6) |
| Compromise Supabase admin (service_role) | ❌ | Out of scope — backend compromise = total game over. Audit + key rotation. |
| Compromise gateway WireGuard | ❌ | Threat di-luar scope. Mitigasi: rotate semua peer keys. |
| MITM antara UI ↔ Supabase | ✅ | HTTPS + (opsional) cert pinning |
| MITM antara UI ↔ Helper (named pipe / unix socket) | ✅ | Lokal kernel — tidak melewati network. Authenticate caller. |
| MITM antara client ↔ gateway WG | ✅ | WireGuard ChaCha20-Poly1305 + Curve25519 + (opsional) PSK |
| Replay attack pada WireGuard handshake | ✅ | Built-in: nonce + counter |
| Steal user JWT dari memory | ⚠️ Partial | TTL 1 jam, refresh rotated, simpan refresh di OS keychain |
| DNS leak | ✅ | DNS dari `wg_config`, plus split-tunnel rules |
| IP leak saat handshake bermasalah | ✅ | Kill switch (lihat §7.7) |

## 7.4 Aliran kunci & data sensitif

| Asset | Disimpan di mana | Kapan ada di memori | Yang boleh akses |
|---|---|---|---|
| WG private key | Client DPAPI / Keychain | Hanya saat inject ke config string sebelum kirim ke Helper | UI app |
| WG public key | Plaintext di `KeyStore`; di-publish ke `user_data.wg_public_key` | Selalu | UI + admin (untuk register peer) |
| PSK | Inside `wg_config` from Supabase | Hanya saat read + relay ke Helper | UI temp + Helper file |
| User JWT | OS keychain encrypted | Saat HTTP request | UI app |
| Supabase service_role key | **TIDAK ADA di client** | N/A | Hanya backend / admin |
| Helper IPC pipe | Kernel-managed | N/A | Process yang punya akses |

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

Kill switch best implemented sebagai field policy di `wg_config` (admin set per user). Helper interpret dan apply firewall rule.

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
| WG keypair | `%LOCALAPPDATA%\HermesNetwork360Guard\Sase\keypair.json` (DPAPI) | `~/Library/Application Support/HermesNetwork360Guard/Sase/keypair.json` (mode 0600) |
| Supabase refresh token | `%LOCALAPPDATA%\HermesNetwork360Guard\session.bin` (DPAPI) | macOS Keychain via `/usr/bin/security` |
| Helper config | N/A (stateless) | N/A |
| WG `.conf` (active) | `C:\Program Files\WireGuard\Data\Configurations\<name>.conf.dpapi.aes` | `/etc/wireguard/<name>.conf` (root:wheel, 0600) |

## 7.10 Audit logging

### 7.10.1 Yang harus di-log

| Event | Lokasi | Retention |
|---|---|---|
| Helper RPC request | Event Viewer (Win) / `os_log` (Mac) | 30 hari |
| Connect / Disconnect | Hermes app log + Supabase `user_data.wg_status` | 30 hari |
| Reconnect attempt | App log | 7 hari |
| Config refresh | App log + Supabase audit | 30 hari |
| Auth failure di Helper | Event Viewer warning | 90 hari |
| `wg_config` change (admin) | Supabase trigger log | indefinite |

### 7.10.2 Yang JANGAN di-log

- WG private key, PSK
- User JWT raw value
- Full WG config (mengandung private key)
- Refresh token
- Supabase service_role / admin tokens

Pattern aman:

```csharp
_log.LogInformation("Helper RPC: {Method} duration_ms={Duration}", method, duration);
// JANGAN: _log.LogInformation("Config: {Config}", configIni);   ❌
```

### 7.10.3 Audit table di Supabase

```sql
CREATE TABLE IF NOT EXISTS sase_audit_log (
    id           BIGSERIAL PRIMARY KEY,
    user_id      UUID NOT NULL REFERENCES auth.users(id),
    action       TEXT NOT NULL,           -- 'connect' | 'disconnect' | 'refresh' | 'error'
    detail       JSONB,                    -- { hostname, public_ip, ... }
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE sase_audit_log ENABLE ROW LEVEL SECURITY;

-- User bisa insert audit untuk dirinya
CREATE POLICY "users_insert_own_audit"
  ON sase_audit_log FOR INSERT
  WITH CHECK (auth.uid() = user_id);

-- User cuma bisa baca milik sendiri (admin lewat service_role)
CREATE POLICY "users_read_own_audit"
  ON sase_audit_log FOR SELECT
  USING (auth.uid() = user_id);
```

Client report:

```csharp
await _http.PostAsync($"{_supabaseUrl}/rest/v1/sase_audit_log",
    JsonContent.Create(new
    {
        action = "connect",
        detail = new { last_handshake = hs, public_ip = await GetPublicIpAsync() }
    }));
```

## 7.11 Key rotation

### 7.11.1 Per-device WG keypair

User-driven rotation kalau curiga compromise:

```csharp
public async Task RotateKeyAsync()
{
    await _conn.DisconnectAsync();
    await _keyStore.DeleteAsync();           // wipe lokal
    // Connect → KeyStore generate baru → push ke Supabase user_data.wg_public_key
    await _conn.ConnectAsync();
    // Admin tooling akan detect public_key change dan re-register peer
}
```

### 7.11.2 PSK

PSK ada di `wg_config` di Supabase. Rotasi = admin update kolom `wg_config` → client refresh otomatis dalam 5 menit.

### 7.11.3 Helper signing certificate

Apple Developer ID + Windows Authenticode cert: rotate sebelum expire. Re-sign + re-deploy installer. Tunnel tetap jalan dengan binary lama sampai user update.

## 7.12 Compliance considerations

| Standar | Yang relevan dari arsitektur ini |
|---|---|
| **SOC 2** | Audit trail di `sase_audit_log`, RLS enforcement, key rotation procedure |
| **ISO 27001** | Threat model di sini, incident response, helper signed binary |
| **HIPAA** | BAA dengan Supabase host, encryption at rest, audit retention |
| **GDPR** | Data minimization (logging policy), right-to-erasure (delete user → cascade audit) |

## 7.13 Disaster recovery

| Skenario | Action |
|---|---|
| Helper Service crash / unresponsive | UI tampilkan banner "Helper not running, please reinstall". Re-register service via installer. |
| Supabase tidak reachable | Cached config terakhir di-pakai (kalau ada). Tunnel tetap up. |
| Gateway WG down | Health monitor detect, status "Faulted". User notif. |
| Compromise gateway | Rotate gateway key + update semua user_data wg_config. Force refresh. |
| User device hilang/dicuri | Admin set `user_data.wg_public_key = null` + revoke peer di gateway. Tunnel drop dalam ~3 menit. |
| Compromise Supabase admin | Rotate service_role key. Audit semua change `wg_config`. |

---

[← Bab 6 Connection Flow]({{ site.baseurl }}{% link docs/06-connection-flow.md %}){: .btn }
[Bab 8 — Dukungan macOS →]({{ site.baseurl }}{% link docs/08-mac-support.md %}){: .btn .btn-primary }
