---
layout: default
title: 9. WireGuard CLI Reference
nav_order: 10
permalink: /docs/cli-reference/
---

# 9. WireGuard CLI Reference
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

Cheat-sheet untuk WireGuard CLI yang dipakai dalam dokumen ini, baik di Windows maupun macOS.

## 9.1 `wg`

Tool inti WireGuard. Tersedia di:
- Windows: `C:\Program Files\WireGuard\wg.exe`
- macOS: `/usr/local/bin/wg`

### Generate keys

```bash
# Private key (base64, 32 bytes)
wg genkey
# Output: bO3GIc8O...=

# Public key (dari private via stdin)
echo <priv-base64> | wg pubkey
# Output: aP5kEN...=

# Pre-shared key
wg genpsk
# Output: x9Yx...=
```

Workflow tipikal:

```bash
priv=$(wg genkey)
pub=$(echo "$priv" | wg pubkey)
psk=$(wg genpsk)
echo "private: $priv"
echo "public:  $pub"
echo "psk:     $psk"
```

### Show interface state

```bash
# All interfaces
wg show

# Specific interface, human format
wg show Hermes

# Output:
# interface: Hermes
#   public key: aP5kEN...
#   private key: (hidden)
#   listening port: 51820
#
# peer: <gateway-pub-key>
#   preshared key: (hidden)
#   endpoint: 35.42.10.5:51820
#   allowed ips: 10.0.0.0/8
#   latest handshake: 12 seconds ago
#   transfer: 5.43 KiB received, 12.21 KiB sent
#   persistent keepalive: every 25 seconds

# Machine-readable (tab-separated)
wg show Hermes dump

# Specific field
wg show Hermes latest-handshakes
wg show Hermes transfer
wg show Hermes endpoints
wg show Hermes peers
```

`dump` format (line 1 = interface, lines 2+ = peers):
```
<priv>  <pub>  <listen-port>  <fwmark>
<peer-pub>  <psk>  <endpoint>  <allowed-ips>  <last-handshake-unix>  <rx-bytes>  <tx-bytes>  <keepalive>
```

### Set / modify peers (advanced)

```bash
# Add peer
sudo wg set Hermes peer <pubkey> \
  preshared-key /path/to/psk-file \
  endpoint 1.2.3.4:51820 \
  allowed-ips 10.0.0.0/24 \
  persistent-keepalive 25

# Remove peer
sudo wg set Hermes peer <pubkey> remove
```

> Untuk Hermes Guard, kita TIDAK pakai `wg set` runtime. Semua mutation lewat `ApplyConfigAsync` yang reload tunnel. Lebih sederhana dan auditable.

## 9.2 `wg-quick`

Wrapper script di atas `wg` yang handle:
- Bring interface up/down
- Set IP address
- Configure DNS
- Setup routing rules

Tersedia di **macOS / Linux**. Windows pakai `wireguard.exe /installtunnelservice` setara.

### Up / Down

```bash
sudo wg-quick up Hermes      # baca /etc/wireguard/Hermes.conf
sudo wg-quick down Hermes
```

`Hermes.conf` minimal:

```ini
[Interface]
PrivateKey = <priv>
Address = 10.99.0.42/32
DNS = 10.0.0.1, 1.1.1.1
MTU = 1280

[Peer]
PublicKey = <gateway-pub>
PresharedKey = <psk>
Endpoint = n1.ndr24.com:51820
AllowedIPs = 10.0.0.0/8
PersistentKeepalive = 25
```

### Konfigurasi advanced

```ini
[Interface]
PrivateKey = ...
Address = 10.99.0.42/32

# Run script saat up/down (kill switch, custom routing)
PostUp   = iptables -A FORWARD -i %i -j ACCEPT
PostDown = iptables -D FORWARD -i %i -j ACCEPT

# Tabel routing custom (untuk policy routing advanced)
Table = off

# fwmark untuk firewall rule integration
FwMark = 0xca6c
```

### Save config saat ini ke file

```bash
sudo wg showconf Hermes > Hermes.snapshot.conf
```

Berguna untuk debug; output **mengandung private key dan PSK**, simpan dengan permission ketat.

## 9.3 `wireguard.exe` (Windows) — REFERENCE SAJA, TIDAK DIPAKAI

> **PENTING:** Hermes Guard memakai **embedded `tunnel.dll`** (lihat `HermesNetwork/TunnelDll/`), **bukan** `wireguard.exe` external. Section ini disimpan sebagai referensi umum untuk troubleshooting manual saat developer ingin reproduksi behavior pakai WireGuard official client di mesin dev. Production code TIDAK panggil `wireguard.exe`.

Satu-satunya tool resmi untuk install/manage tunnel service di Windows.

```powershell
$wg = "C:\Program Files\WireGuard\wireguard.exe"

# Install tunnel sebagai Windows Service (otomatis up + auto-start di reboot)
& $wg /installtunnelservice "C:\path\to\Hermes.conf"

# Uninstall tunnel
& $wg /uninstalltunnelservice "Hermes"

# Open WireGuard GUI (kalau perlu manual debug)
& $wg
```

Setelah `/installtunnelservice`, service `WireGuardTunnel$Hermes` muncul di Service Manager.

```powershell
sc query "WireGuardTunnel`$Hermes"
sc start "WireGuardTunnel`$Hermes"
sc stop  "WireGuardTunnel`$Hermes"
```

> Backtick ` di PowerShell escape dollar sign dalam nama service.

## 9.4 `launchctl` (macOS)

Untuk SASE LaunchDaemon (lihat Bab 8).

```bash
# Modern
sudo launchctl bootstrap system /Library/LaunchDaemons/com.hermesnetwork.sase.plist
sudo launchctl bootout    system/com.hermesnetwork.sase
sudo launchctl kickstart  -k system/com.hermesnetwork.sase    # restart

# Legacy
sudo launchctl load   <plist>
sudo launchctl unload <plist>
sudo launchctl start  <label>
sudo launchctl stop   <label>

# Status
sudo launchctl list com.hermesnetwork.sase
sudo launchctl print system/com.hermesnetwork.sase
```

`launchctl print` lebih informatif daripada `list` di macOS modern; show full state termasuk environment, last exit code, dll.

## 9.5 Diagnostic commands

### Check tunnel up

```bash
# All platforms
wg show

# Berapa peer
wg show Hermes peers | wc -l

# Last handshake (epoch seconds)
wg show Hermes latest-handshakes
```

### Test connectivity via tunnel

```bash
# Ping IP yang ada di AllowedIPs
ping 10.0.0.1

# Cek IP publik (harus IP gateway exit, bukan IP user)
curl https://ifconfig.me

# Cek route
# macOS / Linux
ip route get 8.8.8.8         # Linux
route get 8.8.8.8            # macOS

# Windows
Get-NetRoute -DestinationPrefix "8.8.8.8/32"
```

### Test DNS leak

```bash
# Server resolver yang aktif
# Windows
Get-DnsClientServerAddress

# macOS
scutil --dns | head -20

# Test resolver
nslookup example.com
# Server should be DNS dari config (10.0.0.1), bukan ISP
```

### Test MTU

```bash
# Find effective MTU (kirim packet besar dengan DF flag, kurangi sampai sukses)
# macOS / Linux
ping -M do -s 1372 10.0.0.1   # 1372 + 28 IP/ICMP = 1400 MTU
ping -M do -s 1252 10.0.0.1   # 1280 MTU

# Windows (PowerShell)
Test-Connection 10.0.0.1 -BufferSize 1372
```

WireGuard default MTU = 1420 (Ethernet 1500 - 80 overhead). Kalau pakai VPN over VPN, atau lewat interfaces yang punya overhead lain (PPPoE, GRE), turunkan ke 1280.

## 9.6 Troubleshooting commands

### Tunnel tidak handshake

```bash
# 1. Apakah service up?
# Windows
sc query "WireGuardTunnel`$Hermes"
# macOS
sudo launchctl list com.hermesnetwork.sase

# 2. Apakah UDP 51820 reachable?
nc -uvz n1.ndr24.com 51820

# 3. Apakah ada firewall block?
# Windows
Get-NetFirewallRule | Where DisplayName -Like "*WireGuard*"

# macOS
sudo /usr/libexec/ApplicationFirewall/socketfilterfw --listapps | grep -i wireguard

# 4. Check log
# Windows
Get-EventLog -LogName Application -Source "WireGuard*" -Newest 20

# macOS
tail -50 /var/log/sase-tunnel.err.log
```

### Tunnel up tapi internet tidak jalan

```bash
# Cek apakah AllowedIPs memang covers traffic yang diharapkan
wg show Hermes allowed-ips

# Cek route — apakah trafik benar lewat tunnel
# macOS
route -n get 8.8.8.8 | grep interface
# Should be: interface: utun5 (atau apa pun nama tunnel)

# Cek MTU mismatch
ping -M do -s 1400 8.8.8.8
# Kalau drop, MTU terlalu besar
```

### Bytes counter tidak naik (no traffic)

```bash
# Apakah handshake aktif?
wg show Hermes latest-handshakes
# Jika lebih lama dari 3 menit → koneksi unhealthy

# Manual reconnect
sudo wg-quick down Hermes
sudo wg-quick up Hermes
```

## 9.7 Quick reference table

| Operasi | Windows | macOS |
|---|---|---|
| Generate priv key | `wg genkey` | `wg genkey` |
| Bring tunnel up | `Service.Add(conf, false)` (embedded TunnelDll) + `sc start ...` | `wg-quick up <name>` (binary bundled di app bundle) |
| Bring tunnel down | `sc stop "WireGuardTunnel$Name"` | `wg-quick down <name>` |
| Uninstall tunnel | `Service.Remove(name)` (embedded TunnelDll → `DeleteService` SCM API) | `rm /etc/wireguard/<name>.conf` + bootout LaunchDaemon |
| Show status | `wg show` (cmd) | `wg show` |
| Check service | `sc query "WireGuardTunnel$Name"` | `launchctl list <label>` |
| Logs | Event Viewer | `tail /var/log/sase-tunnel.*.log` |

## 9.8 Useful one-liners

```bash
# Generate complete config from scratch
priv=$(wg genkey); pub=$(echo "$priv" | wg pubkey); psk=$(wg genpsk); cat <<EOF
[Interface]
PrivateKey = $priv
Address = 10.99.0.42/32
DNS = 10.0.0.1
[Peer]
PublicKey = <gateway-pub>
PresharedKey = $psk
Endpoint = n1.ndr24.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

# Watch handshake aktif (Linux/Mac)
watch -n 1 'wg show Hermes latest-handshakes'

# Monitor bytes transferred
watch -n 1 'wg show Hermes transfer'

# Get gateway public key dari running config tunnel
wg show Hermes peers
```

---

[← Bab 8 Dukungan macOS]({{ site.baseurl }}{% link docs/08-mac-support.md %}){: .btn }
[Bab 10 — Rencana Migrasi →]({{ site.baseurl }}{% link docs/10-migrasi.md %}){: .btn .btn-primary }
