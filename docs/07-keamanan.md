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

1. **Private key tidak pernah meninggalkan client device.**
2. **API key SASE control plane hanya di server-side (Supabase Edge Function).**
3. **PSK per-peer** untuk defense in depth.
4. **Identity-aware AllowedIPs** — bukan flat policy untuk semua user.
5. **Audit trail** semua request config + connection event.

## 7.2 Threat model

| Threat | Defended? | Mitigasi |
|---|---|---|
| Steal `.conf` file dari laptop user | ⚠️ Partial | DPAPI (Windows) / mode 0600 (Mac); attacker masih perlu PSK + bypass policy |
| Reverse engineer desktop binary | ✅ | Tidak ada API key SASE / PSK di binary; hanya kode logic |
| Compromise gateway (full takeover) | ❌ | Threat di-luar scope; rotate semua peer key |
| Compromise edge function | ⚠️ | Rotate API key + audit log, scope kerusakan terbatas |
| MITM antara client ↔ Edge Function | ✅ | HTTPS + (opsional) cert pinning |
| MITM antara client ↔ gateway WireGuard | ✅ | Curve25519 ECDH + ChaCha20-Poly1305; PSK additional |
| Replay attack pada WireGuard | ✅ | Built-in: nonce + counter |
| Steal user JWT dari memory | ⚠️ Partial | TTL pendek (1 jam), refresh rotated |
| User compromise (phishing) → enroll device attacker | ⚠️ | RBAC limit damage; admin bisa revoke peer cepat |
| DNS leak | ✅ | DNS dari config + Windows split-tunnel rules |
| IP leak (saat handshake bermasalah) | ✅ | Kill switch (lihat 7.7) |

## 7.3 Aliran kunci & kepercayaan

```
┌────────────────────────────────────────────────────┐
│  Client device                                     │
│   - private_key  (DPAPI / 0600 file)               │
│   - public_key   (di-kirim ke edge function)       │
│   - psk          (di-kirim dari edge function,     │
│                   disimpan di config aktif)        │
│   - user_jwt     (DPAPI / Keychain)                │
└─────────────────┬──────────────────────────────────┘
                  │ HTTPS + Bearer JWT
                  ▼
┌────────────────────────────────────────────────────┐
│  Supabase Edge Function (sase-config)              │
│   - SASE_API_KEY  (env secret)                     │
│   - SASE_GATEWAY_PUBLIC_KEY (env config)           │
└─────────────────┬──────────────────────────────────┘
                  │ HTTPS + API key
                  ▼
┌────────────────────────────────────────────────────┐
│  SASE control plane                                │
│   - All peer pub keys + PSK                        │
└─────────────────┬──────────────────────────────────┘
                  │ deploy peer config
                  ▼
┌────────────────────────────────────────────────────┐
│  SASE gateway (WireGuard)                          │
│   - own private_key                                │
│   - all peer public_key + PSK                      │
└────────────────────────────────────────────────────┘
```

**Yang BOLEH di client device:**
- Private key WireGuard (terkurung dengan DPAPI / mode 0600)
- PSK (sama, terkurung)
- Public key gateway (publik, OK)
- User JWT (terkurung di OS keychain)
- Supabase anon key (publik, dilindungi RLS)

**Yang TIDAK BOLEH di client device:**
- SASE control plane API key
- Supabase service role key
- Gateway private key
- API key TRMM

## 7.4 Pre-Shared Key (PSK)

PSK adalah symmetric secret yang ditambahkan ke handshake WireGuard. Dampak:

- Tanpa PSK: handshake ChaCha20-Poly1305 + Curve25519 ECDH
- Dengan PSK: hash PSK tambahan masuk ke key derivation

Manfaat:
- Defense terhadap **future quantum attack** pada Curve25519 (PSK adalah secret simetrik, kuat terhadap quantum)
- Kalau private key bocor (mis. lewat memory dump), attacker masih perlu PSK untuk handshake
- Memungkinkan rotasi PSK terpisah dari rotasi keypair

Cara generate PSK:

```bash
wg genpsk
# Output: base64 32-byte
```

Di Edge Function (lihat Bab 5), PSK di-generate per-peer di control plane (atau Edge Function bisa generate juga kalau control plane tidak support).

## 7.5 Key rotation

### 7.5.1 Per-device WireGuard keypair

Generate sekali, simpan di `KeyStore`. Rotate kalau:

- User suspect device compromise → reset KeyStore + re-enroll
- Device formatting / OS reinstall → auto-regenerate (KeyStore hilang)
- Annual hygiene rotation

User-driven rotation:

```csharp
public async Task RotateKeyAsync()
{
    // 1. Disconnect current
    await _conn.DisconnectAsync();

    // 2. Wipe KeyStore
    await _keyStore.DeleteAsync();

    // 3. Reconnect — generate keypair baru, register peer baru di control plane
    await _conn.ConnectAsync();

    // 4. (Opsional) Edge function revoke peer lama yang punya pubkey hilang
    //    Implementasikan di edge function: kalau public_key yang masuk berbeda
    //    untuk (user_id, device_id) yang sama, revoke yang lama
}
```

### 7.5.2 PSK rotation

Otomatis tiap config refresh (24 jam default). Tidak ada user action.

### 7.5.3 SASE API key (server-side)

Rotate tiap **6 bulan** atau saat indikasi compromise:

1. Generate API key baru di SASE control plane dashboard
2. `supabase secrets set SASE_API_KEY=<new>`
3. Redeploy edge function
4. Test
5. Delete API key lama

Tidak ada client-side change.

### 7.5.4 Gateway WireGuard private key

Hanya rotate kalau **gateway compromise**. Setelah rotate:

1. Update `SASE_GATEWAY_PUBLIC_KEY` di Supabase secrets
2. Edge function akan return public key baru ke client di config refresh berikutnya
3. Force semua client refresh dalam 5 menit (via flag di edge function "force-refresh-all")

## 7.6 Identity-aware policy

Saat user role berubah di Supabase (mis. `intern` → `engineer`), policy di edge function mengembalikan AllowedIPs baru di config refresh berikutnya. Implementasi:

```typescript
// di handleStatus()
if (profile?.sase_role !== peer.role_at_issue) {
  return jsonOk({ fresh: false, reason: "role-changed" });
}
```

Client polling `IsConfigFreshAsync` tiap 5 menit, dapat `fresh: false` → trigger `RefreshConfigAsync`.

User experience: tidak ada drop koneksi (WireGuard handle peer config update tanpa restart koneksi), AllowedIPs efektif berubah dalam 5–10 menit dari momen role di-update.

### 7.6.1 Revoke akses individual

Untuk admin yang ingin segera cabut akses user (compromise / resign):

```sql
UPDATE sase_peer
SET status = 'revoked'
WHERE user_id = '<user-uuid>';
```

Edge function refuse return config kalau status = revoked → user reconnect = HTTP 403.

Kalau user **sedang connected**: tunggu config TTL expire (max 24 jam) atau force disconnect via:

```typescript
// PUT request di control plane untuk hapus peer
await fetch(`${SASE_API_URL}/peers/${publicKey}`, {
  method: "DELETE",
  headers: { Authorization: `Bearer ${SASE_API_KEY}` }
});
```

Setelah peer dihapus, gateway drop semua trafik → user effectively disconnected dalam < 1 menit.

## 7.7 Kill switch

Saat tunnel **harus** up tapi handshake gagal (gateway down, network filter), kita harus pilih:

- **Soft fail** — tunnel down, semua trafik lewat normal interface
- **Hard fail (kill switch)** — tidak ada trafik kecuali yang lewat tunnel

Untuk role yang sensitive (executive dengan AllowedIPs `0.0.0.0/0`), kill switch wajib.

Cara implement di WireGuard config:

```ini
[Interface]
PrivateKey = ...
Address = 10.99.0.42/32
PostUp   = iptables -I OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT
PreDown  = iptables -D OUTPUT ! -o %i -m mark ! --mark $(wg show %i fwmark) -m addrtype ! --dst-type LOCAL -j REJECT

[Peer]
...
AllowedIPs = 0.0.0.0/0
```

Di Windows pakai feature `BlockUntunneledTraffic` di adapter setting, atau set `AllowedIPs = 0.0.0.0/0` + Windows firewall rule yang block non-tunnel egress.

Kill switch best implemented di Edge Function — tambah field `kill_switch: true` di policy untuk role executive, client interpret saat build wg config.

## 7.8 DNS leak prevention

Set `DNS = ...` di interface block. Untuk Windows, ini install resolver di adapter tunnel. Untuk Mac, `wg-quick` jalankan `resolvconf` setup.

Verifikasi tidak leak:

```bash
# Saat tunnel UP, check DNS server
# Windows
Get-DnsClientServerAddress

# Mac
scutil --dns | head -20

# Test query DNS via tunnel only
nslookup example.com
# Server should be DNS yang di-set di config, BUKAN ISP user
```

Untuk paranoid mode, set firewall rule yang block port 53 ke semua interface kecuali tunnel.

## 7.9 Storage refresh token & PSK

Sudah dibahas di Bab 5 §5.5 (KeyStore). Ringkasan:

| Platform | Storage |
|---|---|
| Windows | `%LOCALAPPDATA%\HermesNetwork360Guard\Sase\keypair.json` (DPAPI) |
| macOS | `~/Library/Application Support/HermesNetwork360Guard/Sase/keypair.json` (mode 0600) |
| (Future) macOS Keychain | `kSecClassGenericPassword` dengan service `com.hermesnetwork.sase` |

PSK disimpan di `.conf` file yang di-apply ke OS service:

| Platform | Config file |
|---|---|
| Windows | `C:\Program Files\WireGuard\Data\Configurations\<name>.conf.dpapi.aes` (DPAPI) |
| macOS | `/etc/wireguard/<name>.conf` (mode 0600 root:wheel) |

## 7.10 Audit logging

### 7.10.1 Yang harus di-log

| Event | Lokasi log | Retention |
|---|---|---|
| Config request / refresh | `sase_audit_log` table | indefinite |
| Connect / disconnect | Hermes app log + Supabase audit | 30 hari |
| Reconnect attempt (auto) | Hermes app log | 7 hari |
| Bytes transferred (periodik) | Supabase audit (sample 1/jam) | 30 hari |
| Peer revoked by admin | `sase_audit_log` + control plane | indefinite |

### 7.10.2 Yang JANGAN di-log

- PrivateKey dalam bentuk apa pun
- PSK
- SASE API key
- Refresh token
- Full WireGuard config (mengandung private key + PSK)

Pattern aman:

```csharp
_log.LogInformation("SASE config refreshed for device {Device} role {Role}",
    deviceId, configDto.Peers[0].AllowedIPs);
// JANGAN: _log.LogInformation("Config: {Config}", configDto);   ❌
```

## 7.11 Edge function security checklist

Sebelum deploy `sase-config` ke produksi:

- [ ] **JWT validation** — semua endpoint validate via `supabase.auth.getUser()`
- [ ] **Service role key** — di-set via `supabase secrets`, bukan kode
- [ ] **SASE API key** — di-set via `supabase secrets`
- [ ] **CORS** — explicit origin di production (Hermes Guard distributable)
- [ ] **Input validation** — sanitize device_id, hostname, public_key
- [ ] **Rate limiting** — Supabase Edge Function default 60 req/min/IP
- [ ] **No secret leakage** — error messages tidak include API key / config
- [ ] **Audit log enable** — `sase_audit_log` populated untuk setiap request
- [ ] **RLS enabled** di `sase_peer` table

## 7.12 Compliance considerations

Kalau Hermes Network 360 Guard tunduk pada standar tertentu:

| Standar | Yang relevan |
|---|---|
| **SOC 2** | Audit trail config request, key rotation policy, access revocation procedure |
| **ISO 27001** | Threat model di sini, incident response untuk gateway compromise |
| **HIPAA** | BAA dengan vendor SASE, encryption at rest untuk audit log |
| **GDPR** | Data minimization (jangan log unnecessary PII), right-to-erasure flow |

## 7.13 Disaster recovery

### 7.13.1 SASE control plane down

→ Existing tunnel keep working sampai TTL expire (24 jam). User baru tidak bisa enroll. Restart control plane → semua kembali normal.

### 7.13.2 Edge function compromise

→ Rotate `SASE_API_KEY` immediately, audit `sase_audit_log` untuk indikasi abuse, force disconnect semua peer yang dicurigai.

### 7.13.3 Gateway hardware failure

→ Failover ke gateway backup (kalau ada). Update `SASE_GATEWAY_ENDPOINT` di Supabase secrets, semua client auto-refresh dalam 5 menit.

### 7.13.4 KeyStore client hilang

→ User reconnect → KeyStore kosong → generate keypair baru → register peer baru. Lama (sebelumnya) di-revoke otomatis di edge function (lihat 7.5.1).

---

[← Bab 6 Connection Flow]({{ site.baseurl }}{% link docs/06-connection-flow.md %}){: .btn }
[Bab 8 — Dukungan macOS →]({{ site.baseurl }}{% link docs/08-mac-support.md %}){: .btn .btn-primary }
