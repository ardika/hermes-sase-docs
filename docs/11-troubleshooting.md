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
# All platforms
wg show
wg show Hermes latest-handshakes
wg show Hermes transfer
ping <internal-IP-yang-harus-jangkau-via-tunnel>
curl https://ifconfig.me   # IP harus IP gateway, bukan IP user

# Edge function health
curl -X POST $SB_URL/functions/v1/sase-config \
  -H "Authorization: Bearer $JWT" \
  -d '{"device_id":"test","public_key":"<pub>","platform":"windows"}'
```

Lokasi log:

| Apa | Windows | macOS |
|---|---|---|
| Hermes app log | `%LOCALAPPDATA%\HermesNetwork360Guard\app.log` | `~/Library/Logs/HermesNetwork360Guard/app.log` |
| WireGuard tunnel log | Event Viewer → Application (source: WireGuard) | `/var/log/sase-tunnel.{out,err}.log` |
| Edge function log | Supabase Dashboard → Functions → Logs | sama |

## 11.2 "Tunnel tidak handshake"

**Gejala:** `wg show Hermes` menunjukkan peer ada, tapi `latest-handshake` kosong atau "X seconds ago" tidak update.

| Penyebab | Cek | Fix |
|---|---|---|
| Firewall block UDP 51820 outbound | `nc -uvz n1.ndr24.com 51820` | Whitelist gateway endpoint di firewall corporate |
| Public key mismatch (client → gateway) | Compare `wg show` peer pubkey ↔ value di SASE control plane | Refresh config, pastikan edge function pakai `SASE_GATEWAY_PUBLIC_KEY` yang benar |
| PSK mismatch | (tidak ada cara langsung; check via try without PSK) | Refresh config |
| Gateway down | curl status page gateway | Hubungi ops |
| Endpoint IP berubah | Compare config Endpoint vs DNS resolve `n1.ndr24.com` | `wg-quick down && up` (DNS re-resolved) |
| MTU terlalu besar (packet drop) | `ping -M do -s 1372 <gw>` | Set `MTU = 1280` di config |
| NAT timeout | Tidak ada keepalive | Set `PersistentKeepalive = 25` |

**Diagnosa step-by-step:**

```bash
# 1. Apakah service running?
# Windows
sc query "WireGuardTunnel`$Hermes"
# macOS
sudo launchctl list com.hermesnetwork.sase

# 2. Apakah config benar di-loaded?
# Windows
type "C:\Program Files\WireGuard\Data\Configurations\Hermes.conf.dpapi.aes"   # binary, butuh decrypt
# macOS
sudo cat /etc/wireguard/Hermes.conf

# 3. UDP test
# All
nc -uvz n1.ndr24.com 51820

# 4. Tcpdump (Mac/Linux) untuk lihat WireGuard packet
sudo tcpdump -i any -n udp port 51820 -vv

# 5. Manual reconnect
sudo wg-quick down Hermes
sudo wg-quick up Hermes
sleep 5
wg show Hermes latest-handshakes
```

## 11.3 "Tunnel up, tapi internet tidak jalan"

**Gejala:** `wg show` menunjukkan handshake fresh, tapi browser/curl tidak jalan.

| Penyebab | Cek | Fix |
|---|---|---|
| AllowedIPs tidak cover destination | `wg show Hermes allowed-ips` | Edit policy di edge function role |
| Routing tidak masuk ke tunnel interface | `route -n get 8.8.8.8` (Mac) | `wg-quick` biasanya handle; restart tunnel |
| MTU drop packet | `ping -M do -s 1400 8.8.8.8` | Set MTU 1280 |
| Gateway block specific traffic (corporate policy) | Test ke IP allowed dulu | Hubungi ops |
| DNS leak / DNS broken | `nslookup example.com` | Set DNS di config, atau `dig @10.0.0.1 example.com` |

## 11.4 "Connection refused" saat call edge function

**Gejala:** `RequestConfigAsync` throw `HttpRequestException`.

| Penyebab | Fix |
|---|---|
| Supabase URL salah | Verify env var `HNGUARD_SUPABASE_URL` |
| Edge function tidak deployed | `supabase functions deploy sase-config` |
| Edge function crash | Check Supabase logs, fix bug |
| User JWT invalid / expired | Re-login user, refresh token |

## 11.5 "InvalidApiKey" dari edge function

**Gejala:** Edge function return 401 / 403 ke client.

```bash
# Check log Supabase
supabase functions logs sase-config --tail
```

Cari pesan error spesifik. Biasanya:

- `Invalid token` → JWT user invalid, refresh user session
- `SASE register failed: 401` → SASE_API_KEY di Supabase salah
- `SASE register failed: 404` → endpoint API SASE salah

**Fix SASE_API_KEY:**

```bash
supabase secrets set SASE_API_KEY=<correct-key>
supabase functions deploy sase-config
```

## 11.6 "DPAPI failed to decrypt KeyStore"

**Gejala:** App throw `CryptographicException` saat baca `keypair.json` di Windows.

**Penyebab:**
- File di-copy dari user lain (DPAPI tied to user account)
- User profile rebuilt
- Windows reinstall

**Fix:** Delete file, generate keypair baru.

```powershell
Remove-Item "$env:LOCALAPPDATA\HermesNetwork360Guard\Sase\keypair.json"
# Klik Connect lagi → akan auto-generate keypair baru
```

## 11.7 macOS: "Operation not permitted" saat start tunnel

**Gejala:** osascript exit dengan "Operation not permitted" atau LaunchDaemon tidak load.

| Cause | Check | Fix |
|---|---|---|
| App belum signed/notarized | `codesign --verify --deep <app>` | Re-sign + notarize |
| LaunchDaemon plist permission salah | `ls -la /Library/LaunchDaemons/com.hermesnetwork.sase.plist` | `sudo chown root:wheel`, `chmod 644` |
| User cancel password dialog | (no specific log) | Re-attempt, edukasi user |
| SIP block | Check System Integrity Protection | Tidak relevan untuk binary di /usr/local/, harusnya OK |

## 11.8 "Tunnel disconnect tiap 25 detik"

**Gejala:** Tunnel up briefly, lalu drop, ulang setiap 25 detik.

**Penyebab umum:** PSK mismatch. Handshake awal sukses karena WireGuard bisa fall back ke "no PSK" mode pada beberapa implementasi tua, tapi gateway expect PSK.

**Fix:**
1. Refresh config dari edge function
2. Verify config baru ada PSK line
3. Reconnect

## 11.9 "Bytes counter tidak naik (no traffic)"

**Gejala:** Handshake terlihat fresh, tapi `transfer: 0 B received, 0 B sent` selama menit.

**Diagnosa:**

```bash
# Apakah ada apps yang push traffic?
# Test ping ke IP yang ada di AllowedIPs
ping 10.0.0.1

# Apakah keepalive jalan?
wg show Hermes persistent-keepalive
# Should: 25 seconds

# Tcpdump
sudo tcpdump -i utun5 -n   # Mac (atau nama tunnel kamu)
```

Kalau `tcpdump` show no packet di interface tunnel → routing salah. Kalau show packet tapi reply tidak datang → gateway-side issue.

## 11.10 macOS: "Client request edge function gagal — TLS error"

**Gejala:** `HttpRequestException: The SSL connection could not be established`.

**Penyebab:** macOS ATS (App Transport Security) block koneksi non-HTTPS atau cert tidak trusted.

**Fix:**
- Pastikan endpoint pakai HTTPS valid (Let's Encrypt OK)
- Kalau staging pakai self-signed cert, tambah exception di `Info.plist`:

```xml
<key>NSAppTransportSecurity</key>
<dict>
  <key>NSExceptionDomains</key>
  <dict>
    <key>staging.supabase.co</key>
    <dict>
      <key>NSExceptionAllowsInsecureHTTPLoads</key>
      <true/>
    </dict>
  </dict>
</dict>
```

> JANGAN pakai exception di production — selalu HTTPS valid.

## 11.11 "Reconnect loop" — tunnel up-down terus

**Gejala:** UI status flicker antara "Connecting" dan "Reconnecting".

**Penyebab:**
- Config invalid (salah satu field) → start gagal → monitor detect → reconnect → ulang
- Gateway intermittent
- NAT terlalu agresif

**Diagnosa:**

```bash
# Cek log Hermes app
# cari pattern "SASE handshake stale" atau "Reconnect timeout"

# Manual test stable connection (bypass monitor):
sudo wg-quick down Hermes
sudo wg-quick up Hermes
# Tunggu 60 detik sambil
watch -n 1 'wg show Hermes latest-handshakes'
```

Kalau manual stable tapi via app tidak → ada masalah di logic monitor service. Tambah logging detail.

## 11.12 Performa: tunnel bandwidth rendah

**Gejala:** Speed via tunnel jauh di bawah speed normal.

**Penyebab umum:**
- MTU terlalu besar / kecil
- CPU encryption bottleneck (di laptop low-end)
- Gateway overload
- Routing path tidak optimal

**Test:**

```bash
# Speedtest tanpa tunnel
speedtest-cli

# Up tunnel + speedtest lagi
sudo wg-quick up Hermes
speedtest-cli
```

Kalau drop > 50%, suspect MTU atau CPU. Test dengan MTU 1280:

```ini
[Interface]
MTU = 1280
```

## 11.13 "Config refresh tidak trigger"

**Gejala:** Admin update user role, tapi user tidak terapply policy baru.

**Diagnosa:**

```bash
# 1. Check status di edge function
curl "$SB_URL/functions/v1/sase-config/status?device=<id>" \
  -H "Authorization: Bearer $JWT"
# Expected: { fresh: false, reason: "role-changed" }
```

Kalau output `fresh: true` → user_profiles.sase_role belum di-update. Kalau `fresh: false` tapi client tidak refresh → masalah di monitor loop:

```
# Cek log app — apakah background monitor active?
# Kalau tab SASE belum dibuka, monitor mungkin tidak start
```

**Fix:**
- Pastikan user buka tab SASE atau force `RefreshConfigAsync()` dipanggil saat login
- Tambah broadcast notification dari Supabase Realtime kalau role berubah

## 11.14 Build / sign issues macOS

| Issue | Fix |
|---|---|
| `notarytool` reject "Hardened runtime not enabled" | Re-sign dengan `--options runtime` |
| `notarytool` reject "Code object is not signed at all" | Sign semua dylib + executable di bundle |
| `spctl` reject signed app | Run `xcrun stapler staple` setelah notarization sukses |
| Universal binary tidak jalan di Intel Mac | Verify lipo: `lipo -info HermesNetwork360Guard` |

## 11.15 Diagnostic checklist untuk support ticket

Saat user lapor masalah SASE, kumpulkan:

- [ ] OS + version
- [ ] Hermes Network 360 Guard version
- [ ] Username / email Supabase
- [ ] Hostname endpoint
- [ ] Output `wg show Hermes` (atau `wg show` saja)
- [ ] Output `wg show Hermes latest-handshakes`
- [ ] Output `nc -uvz n1.ndr24.com 51820`
- [ ] Tail 100 lines `app.log` (decrypted kalau encrypted)
- [ ] Tail 50 lines tunnel log
- [ ] Screenshot UI saat error
- [ ] Steps to reproduce

Bikin command shortcut "diagnose-sase" yang auto-collect ini ke zip.

## 11.16 Eskalasi

| Skenario | Eskalasi ke |
|---|---|
| WireGuard kernel module crash | wireguard-tools upstream |
| MeshCentral / TRMM tidak relevan dengan SASE — beda doc |
| Edge function bug | Internal repo Hermes |
| Bug di TunnelSupervisor | Internal repo Hermes |
| Gateway issue | Vendor SASE / ops Hermes |
| Bug di Avalonia | [github.com/AvaloniaUI/Avalonia](https://github.com/AvaloniaUI/Avalonia) |

---

[← Bab 10 Rencana Migrasi]({{ site.baseurl }}{% link docs/10-migrasi.md %}){: .btn }
[Bab 12 — FAQ →]({{ site.baseurl }}{% link docs/12-faq.md %}){: .btn .btn-primary }
