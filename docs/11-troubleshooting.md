---
layout: default
title: 11. Troubleshooting
nav_order: 12
permalink: /docs/troubleshooting/
---

# 11. Troubleshooting
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 11.1 Diagnostic toolkit

```bash
# Lokal (Win / Mac sama)
wg show
wg show Hermes latest-handshakes
wg show Hermes transfer
ping <internal-IP-yang-ada-di-AllowedIPs>
curl https://ifconfig.me   # IP harus IP gateway

# Helper alive check (named pipe / unix-socket)
# Kirim Ping JSON-RPC manual via test program

# Supabase query test
curl "$SUPABASE_URL/rest/v1/user_data?select=wg_config" \
  -H "apikey: $ANON" -H "Authorization: Bearer $JWT"
```

Lokasi log:

| Apa | Windows | macOS |
|---|---|---|
| Hermes UI app | `%LOCALAPPDATA%\HermesNetwork360Guard\app.log` | `~/Library/Logs/HermesNetwork360Guard/app.log` |
| Hermes Helper | Event Viewer → Application (Source: HermesHelperSvc) | `/var/log/hermes-helper.{out,err}.log` |
| WireGuard tunnel | Event Viewer → Application (Source: WireGuard) | (none — Helper log) |

## 11.2 Helper Service tidak respond

**Gejala:** `_helper.PingAsync()` throw timeout atau "pipe not found".

| Penyebab | Cara cek | Fix |
|---|---|---|
| Helper Service belum terinstall | `sc query HermesHelperSvc` (Win) / `launchctl list com.hermesnetwork.helper` (Mac) | Reinstall via installer |
| Helper Service crash | Event Viewer / `/var/log/hermes-helper.err.log` | Cek error, fix bug, restart |
| Pipe / socket permission salah | Win: `accesschk -p HermesHelperSvc` / Mac: `ls -la /var/run/hermes-helper.sock` | Restart Helper, pastikan permission setup di startup |
| UI binary tidak signed | Helper authentication reject | Sign UI binary dengan cert yang sama |

**Diagnosa step-by-step:**

```powershell
# Windows
sc query HermesHelperSvc
# Expected: STATE: 4 RUNNING

Get-EventLog -LogName Application -Source HermesHelperSvc -Newest 20

# Test pipe manual
$pipe = New-Object IO.Pipes.NamedPipeClientStream(".", "HermesHelper", "InOut")
$pipe.Connect(2000)   # 2-second timeout
```

```bash
# macOS
sudo launchctl list com.hermesnetwork.helper
# Look for "PID" > 0

sudo tail -50 /var/log/hermes-helper.err.log

# Test socket
nc -U /var/run/hermes-helper.sock < /dev/null
```

## 11.3 Tunnel tidak handshake

**Gejala:** `wg show` peer ada, tapi `latest-handshake` kosong / sangat lama.

| Penyebab | Cek | Fix |
|---|---|---|
| Firewall block UDP 51820 outbound | `nc -uvz <gateway> 51820` | Whitelist gateway endpoint |
| Public key client tidak ter-register di gateway | Compare `wg_public_key` di Supabase ↔ pubkey di gateway peer list | Trigger re-register: clear `wg_public_key` di Supabase, reconnect |
| PSK mismatch | (tidak ada cara langsung) | Refresh config dari Supabase |
| Endpoint IP berubah / DNS resolve gagal | `nslookup <gateway-host>` | `wg-quick down && up` (force re-resolve) |
| MTU terlalu besar | `ping -M do -s 1372 <gw>` | Set `MTU = 1280` di config |
| NAT timeout | Tidak ada keepalive | `PersistentKeepalive = 25` |

**Diagnosa:**

```bash
# UDP test
nc -uvz <gateway-host> 51820

# Tcpdump (Mac/Linux)
sudo tcpdump -i any -n udp port 51820 -vv

# Manual reconnect
sudo wg-quick down /etc/wireguard/Hermes.conf
sudo wg-quick up /etc/wireguard/Hermes.conf
```

## 11.4 Tunnel up tapi internet tidak jalan

| Penyebab | Cek | Fix |
|---|---|---|
| AllowedIPs tidak cover destination | `wg show Hermes allowed-ips` | Edit `wg_config` di Supabase |
| Routing tidak ke tunnel interface | `route get 8.8.8.8` (Mac) / `Get-NetRoute` (Win) | Restart tunnel |
| MTU drop packet | `ping -M do -s 1400 8.8.8.8` | Set MTU 1280 |
| DNS broken | `nslookup example.com` | Verify DNS line di config |
| Gateway-side block | Test ke IP allowed dulu | Hubungi ops |

## 11.5 SaseConfigClient: "wg_config kosong"

**Gejala:** `GetConfigAsync()` throw "wg_config kosong di user_data".

**Penyebab:** Tabel `user_data` row user belum di-populate dengan `wg_config`.

**Fix:**

1. Verify dengan query:
   ```sql
   SELECT id, length(wg_config) FROM user_data WHERE id = '<user-uuid>';
   ```
2. Kalau NULL → koordinasi dengan ops untuk populate config
3. Pastikan tooling admin generate per-user config + save ke Supabase

## 11.6 SaseConfigClient: "user_data row not found"

**Penyebab:**
- User profile belum ter-create
- RLS policy salah
- JWT user invalid / expired

**Fix:**

```sql
-- Verify row exists
SELECT * FROM user_data WHERE id = '<user-uuid>';

-- Verify RLS policy
SELECT polname, qual FROM pg_policies WHERE tablename = 'user_data';
```

Kalau row hilang, create dulu (via trigger di `auth.users` atau manual):

```sql
INSERT INTO user_data (id) VALUES ('<user-uuid>') ON CONFLICT DO NOTHING;
```

## 11.7 Helper RPC error code -32001 ("Caller not authenticated")

**Penyebab:** Caller authenticator reject — caller tidak punya identity yang valid.

**Diagnosa:**
- Apakah UI binary signed dengan cert yang Helper trust?
- Apakah caller running dengan user identity yang dikenal?

**Fix:**
- Sign ulang UI binary dengan cert yang sama dengan Helper
- Verify policy di `CallerAuthenticator` (mungkin terlalu ketat)

## 11.8 Helper RPC error -32000 (validation failed)

**Penyebab:** Input gagal validation:
- Tunnel name regex mismatch
- Config tidak ada `[Interface]` atau `[Peer]` section
- Config terlalu besar (>16 KiB)

**Fix:**
- Cek config string yang di-pass ke `ApplyConfig`
- Pastikan format INI valid (kalau pakai JSON di Supabase, parser converted ke INI)
- Log config dengan PrivateKey + PSK redacted untuk debug

## 11.9 Helper RPC error -32002 (OS operation failed)

**Penyebab:** Command OS gagal — `wireguard.exe` exit non-zero, `wg-quick` error, `sc` access denied, dll.

**Diagnosa:** Lihat `error.data` di response — biasanya forward stderr dari command yang gagal.

**Fix tergantung error message:**
- "Access denied" → Helper tidak running as SYSTEM/root → reinstall service
- "File not found: wireguard.exe" → WG belum installed
- "Service already exists" → uninstall existing tunnel dulu sebelum install ulang

## 11.10 Performance: tunnel bandwidth rendah

**Gejala:** Speed via tunnel jauh di bawah speed normal.

```bash
speedtest-cli                            # tanpa tunnel
sudo wg-quick up /etc/wireguard/Hermes
speedtest-cli                            # dengan tunnel
```

| Cause | Fix |
|---|---|
| MTU mismatch | Set `MTU = 1280` |
| CPU encryption (laptop low-end) | (hardware limit) |
| Gateway overload | Hubungi ops |
| Routing path tidak optimal | Multi-region gateway selection (advanced) |

## 11.11 macOS: notarization issues

| Issue | Fix |
|---|---|
| Reject "Hardened runtime not enabled" | Re-sign dengan `--options runtime` |
| Reject "Code object is not signed at all" | Sign semua dylib + executable + Helper binary di bundle |
| Reject "Invalid signature" | Verify identity, regenerate p12 kalau perlu |
| `spctl` reject signed app | Run `xcrun stapler staple` setelah notarization |

## 11.12 Reconnect loop

**Gejala:** UI status flicker antara Connecting/Reconnecting/Faulted.

**Penyebab:** Config tidak valid, gateway intermittent, atau monitor logic terlalu agresif.

**Diagnosa:**

```
# Manual stable test (bypass monitor):
sudo wg-quick down /etc/wireguard/Hermes
sudo wg-quick up /etc/wireguard/Hermes
watch -n 1 'wg show Hermes latest-handshakes'
```

Kalau manual stable tapi via app tidak → bug di logic `MonitorLoopAsync`. Tambah verbose logging.

## 11.13 KeyStore corrupt (Win DPAPI)

**Gejala:** `KeyStore.LoadAsync` throw `CryptographicException`.

**Penyebab:**
- File di-copy dari user lain (DPAPI tied to user account)
- User profile rebuilt
- Windows reinstall

**Fix:** Hapus file, generate ulang.

```powershell
Remove-Item "$env:LOCALAPPDATA\HermesNetwork360Guard\Sase\keypair.json"
# UI klik Connect lagi → generate keypair baru
# Public key baru di-publish ke user_data → admin tooling re-register
```

## 11.14 "Config refresh tidak trigger"

**Gejala:** Admin update `wg_config` di Supabase, tapi user tidak terapply.

**Diagnosa:**
- Apakah Realtime subscription aktif?
- Apakah polling 5 menit jalan?

**Fix:**
- Pastikan tab SASE dibuka (monitor active hanya saat connected)
- Tambah broadcast notification dari trigger Supabase
- Atau force user RefreshConfig manual

## 11.15 Diagnostic checklist untuk support ticket

- [ ] OS + version
- [ ] Hermes Network 360 Guard version
- [ ] Helper Service version (`HermesHelperSvc --version`)
- [ ] Username Supabase
- [ ] Hostname endpoint
- [ ] `wg show Hermes` output
- [ ] Helper service status (Win: `sc query` / Mac: `launchctl list`)
- [ ] Tail 100 lines `app.log` (decrypted kalau ter-encrypt)
- [ ] Tail 50 lines Helper error log
- [ ] Steps to reproduce
- [ ] Screenshot

## 11.16 Eskalasi

| Skenario | Eskalasi ke |
|---|---|
| WireGuard kernel module crash | wireguard-tools upstream |
| Helper Service bug | Internal repo HermesHelperSvc |
| Bug client `SaseConnectionService` / `SaseConfigClient` | Internal repo HermesNetwork360Guard |
| Schema `user_data` issue | DBA + ops |
| Gateway issue | Vendor SASE / ops Hermes |
| Bug Avalonia | [github.com/AvaloniaUI/Avalonia](https://github.com/AvaloniaUI/Avalonia) |

---

[← Bab 10 Rencana Migrasi]({{ site.baseurl }}{% link docs/10-migrasi.md %}){: .btn }
[Bab 12 — FAQ →]({{ site.baseurl }}{% link docs/12-faq.md %}){: .btn .btn-primary }
