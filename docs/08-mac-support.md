---
layout: default
title: 8. Dukungan macOS
nav_order: 9
permalink: /docs/mac-support/
---

# 8. Dukungan macOS
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 8.1 Dua opsi WireGuard di macOS

| Opsi | Deskripsi | Cocok untuk |
|---|---|---|
| **WireGuard.app** dari Mac App Store | UI resmi, NetworkExtension, sandboxed | End-user manual install |
| **wireguard-go + wg-quick** | CLI, jalan sebagai LaunchDaemon | Otomasi enterprise |

Untuk Hermes Network 360 Guard yang butuh full automation, pakai **wireguard-go + wg-quick**.

## 8.2 Distribusi WireGuard binaries

Cara paling clean: bundle binary `wg`, `wg-quick`, dan `wireguard-go` ke dalam `.pkg` installer Hermes Guard, dan install ke `/usr/local/bin/`.

```
Hermes-Network-360-Guard.pkg
└── Payload/
    ├── Applications/
    │   └── HermesNetwork360Guard.app/
    └── usr/local/bin/
        ├── wg
        ├── wg-quick
        └── wireguard-go
```

Source binary dari project resmi: [git.zx2c4.com/wireguard-tools](https://git.zx2c4.com/wireguard-tools/) dan [git.zx2c4.com/wireguard-go](https://git.zx2c4.com/wireguard-go/). Build sendiri (recommended) atau ambil dari Homebrew bottle.

```bash
# Build wg + wg-quick
git clone https://git.zx2c4.com/wireguard-tools
cd wireguard-tools/src
make
sudo make install   # ke /usr/local/bin/
```

Verifikasi installed:

```bash
which wg            # /usr/local/bin/wg
which wg-quick      # /usr/local/bin/wg-quick
wg --version        # wireguard-tools v1.0.20210914
```

## 8.3 LaunchDaemon untuk SASE tunnel

LaunchDaemon (di `/Library/LaunchDaemons/`) jalan sebagai root, persistent across reboot, dan tidak butuh user login.

`/Library/LaunchDaemons/com.hermesnetwork.sase.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.hermesnetwork.sase</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/wg-quick</string>
        <string>up</string>
        <string>Hermes</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <false/>

    <key>StandardOutPath</key>
    <string>/var/log/sase-tunnel.out.log</string>

    <key>StandardErrorPath</key>
    <string>/var/log/sase-tunnel.err.log</string>

    <key>UserName</key>
    <string>root</string>

    <key>GroupName</key>
    <string>wheel</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    </dict>
</dict>
</plist>
```

Permission: `chown root:wheel`, `chmod 644`.

### 8.3.1 Manage via launchctl

```bash
# Modern API (macOS 10.10+, recommended)
sudo launchctl bootstrap system /Library/LaunchDaemons/com.hermesnetwork.sase.plist
sudo launchctl bootout    system/com.hermesnetwork.sase

# Force restart
sudo launchctl kickstart -k system/com.hermesnetwork.sase

# Check status
sudo launchctl list com.hermesnetwork.sase
```

`launchctl kickstart -k system/<label>` adalah cara terbaik untuk apply config baru: stop dulu, baru start ulang dalam satu command.

### 8.3.2 Legacy API (kalau butuh kompat)

```bash
sudo launchctl load   /Library/LaunchDaemons/com.hermesnetwork.sase.plist
sudo launchctl unload /Library/LaunchDaemons/com.hermesnetwork.sase.plist
sudo launchctl start  com.hermesnetwork.sase
sudo launchctl stop   com.hermesnetwork.sase
```

## 8.4 Code signing aplikasi Hermes

Untuk distribusi di luar App Store, butuh **Developer ID Application** + notarization:

### 8.4.1 Apple Developer ID

```bash
# Verify identity tersedia
security find-identity -v -p codesigning
# Output baris: "Developer ID Application: Hermes Network Inc. (XXXXXXXXXX)"
```

### 8.4.2 Sign binary WireGuard yang dibundle

```bash
APP_PATH="bin/Release/net8.0/osx-arm64/publish/HermesNetwork360Guard.app"
SIGN_ID="Developer ID Application: Hermes Network Inc. (XXXXXXXXXX)"

# Sign WireGuard binaries dulu (kalau di-bundle di .app/Contents/MacOS/wg etc.)
for bin in "$APP_PATH/Contents/MacOS/wg" \
           "$APP_PATH/Contents/MacOS/wg-quick" \
           "$APP_PATH/Contents/MacOS/wireguard-go"; do
  codesign --force --options runtime --sign "$SIGN_ID" --timestamp "$bin"
done

# Sign all .dylib + .so + executables
find "$APP_PATH" -type f \( -name "*.dylib" -o -name "*.so" -o -perm +111 \) \
  -exec codesign --force --options runtime --sign "$SIGN_ID" --timestamp {} \;

# Sign top-level bundle dengan entitlements
codesign --force --options runtime --sign "$SIGN_ID" --timestamp \
  --entitlements "Resources/HermesNetwork360Guard.entitlements" \
  "$APP_PATH"

# Verify
codesign --verify --deep --strict --verbose=2 "$APP_PATH"
spctl -a -v "$APP_PATH"
```

### 8.4.3 Entitlements

`HermesNetwork360Guard.entitlements`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.network.server</key>
    <false/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
    <!-- Untuk memanggil wg-quick / launchctl via osascript -->
    <key>com.apple.security.automation.apple-events</key>
    <true/>
</dict>
</plist>
```

`Info.plist` perlu privacy usage description:

```xml
<key>NSAppleEventsUsageDescription</key>
<string>Hermes Network 360 Guard butuh elevasi untuk memasang & menjalankan SASE tunnel.</string>
```

### 8.4.4 Notarization

```bash
ditto -c -k --keepParent "$APP_PATH" HermesNetwork360Guard.zip

xcrun notarytool submit HermesNetwork360Guard.zip \
  --keychain-profile AC_PASSWORD \
  --wait

xcrun stapler staple "$APP_PATH"
xcrun stapler validate "$APP_PATH"
```

`AC_PASSWORD` keychain profile setup sekali:

```bash
xcrun notarytool store-credentials AC_PASSWORD \
  --apple-id "your-apple-id@hermesnetwork.com" \
  --team-id "XXXXXXXXXX" \
  --password "abcd-efgh-ijkl-mnop"
```

## 8.5 osascript untuk privilege elevation

`MacTunnelSupervisor` di Bab 4 pakai osascript:

```bash
osascript -e 'do shell script "wg-quick up Hermes" with administrator privileges'
```

Yang terjadi:

1. macOS munculkan dialog: *"Hermes Network 360 Guard wants to make changes."*
2. User input password admin
3. Shell command jalan sebagai root
4. Dialog cuma muncul **sekali per session app**

**Kekurangan:**

- User experience: dialog muncul setiap kali start aplikasi (session baru). Mitigasi: minimal call ke elevation, batch operation.
- Tidak ada cara pre-approve di code — selalu user interaction.

**Best practice:** prompt elevation di **awal** flow (saat user klik "Connect SASE pertama kali"), simpan koneksi dalam memory, sehingga sesi berikutnya gabung.

## 8.6 Privacy permissions (TCC)

WireGuard tidak butuh Full Disk Access atau Screen Recording. Tapi mungkin butuh:

| Resource | Kapan butuh |
|---|---|
| Network access | Otomatis granted untuk app signed |
| File access (config /etc/wireguard) | Tidak butuh TCC — pakai sudo via osascript |

User tidak akan lihat dialog TCC selama scope-nya hanya networking.

## 8.7 Mac firewall (Application Firewall)

macOS Application Firewall block listener inbound by default. WireGuard hanya bikin **outbound UDP** ke gateway, jadi tidak ada dialog firewall.

Verifikasi tidak ada listener:

```bash
sudo lsof -nP -iUDP | grep -i wireguard
# Output: hanya outbound koneksi
```

## 8.8 NetworkExtension vs LaunchDaemon

Apple modern recommendation untuk VPN di macOS adalah **NetworkExtension framework** (kernel-mode VPN, lebih efisien). Tapi:

- NetworkExtension butuh **Special Entitlement** dari Apple yang sulit didapat untuk app kustom enterprise
- WireGuard.app dari App Store pakai NetworkExtension; binary `wireguard-go` adalah userspace fallback yang tetap kerja

Untuk Hermes Network scale, **LaunchDaemon + wireguard-go cukup**. Migrasi ke NetworkExtension = future work kalau butuh:

- Kernel-mode WireGuard (lebih cepat)
- Always-on VPN profile yang di-enforce MDM
- Per-app VPN

## 8.9 Build matrix

```bash
# Universal binary untuk Mac (Intel + ARM)
dotnet publish -c Release -r osx-arm64 --self-contained -o publish/arm64
dotnet publish -c Release -r osx-x64   --self-contained -o publish/x64

# Lipo
lipo -create publish/arm64/HermesNetwork360Guard \
            publish/x64/HermesNetwork360Guard \
     -output publish/universal/HermesNetwork360Guard

# Repack ke .app universal
```

## 8.10 PKG installer

Buat installer `.pkg` yang:

1. Install `HermesNetwork360Guard.app` ke `/Applications/`
2. Install WireGuard binaries ke `/usr/local/bin/`
3. (Optional) Install LaunchDaemon plist ke `/Library/LaunchDaemons/` (untuk auto-start tunnel kalau di-config later)

```bash
# Component pkg
pkgbuild --root "publish-staging" \
         --identifier "com.hermesnetwork.guard" \
         --version "8.4.0" \
         --install-location "/" \
         --scripts "scripts/" \
         HermesNetwork360Guard.component.pkg

# Sign
productsign --sign "Developer ID Installer: Hermes Network Inc. (XXXXXXXXXX)" \
            HermesNetwork360Guard.component.pkg \
            HermesNetwork360Guard.pkg

# Notarize + staple
xcrun notarytool submit HermesNetwork360Guard.pkg --keychain-profile AC_PASSWORD --wait
xcrun stapler staple HermesNetwork360Guard.pkg
```

`scripts/postinstall`:

```bash
#!/bin/bash
set -e

# Set permission yang benar untuk binary
chmod 755 /usr/local/bin/wg /usr/local/bin/wg-quick /usr/local/bin/wireguard-go
chown root:wheel /usr/local/bin/wg /usr/local/bin/wg-quick /usr/local/bin/wireguard-go

# Buat directory wireguard
mkdir -p /etc/wireguard
chmod 700 /etc/wireguard

# Buat log directory
mkdir -p /var/log
touch /var/log/sase-tunnel.out.log /var/log/sase-tunnel.err.log
chmod 644 /var/log/sase-tunnel.*.log

exit 0
```

## 8.11 Testing checklist macOS

- [ ] Build `.app` di mesin Mac (atau CI dengan macOS runner)
- [ ] Sign + notarize sukses, `spctl -a -v` return "accepted"
- [ ] Install di Mac bersih (clean VM atau wipe)
- [ ] Login dengan user non-admin
- [ ] Klik "Connect SASE" → osascript dialog muncul → user input password admin
- [ ] WireGuard tunnel up: `wg show Hermes` menunjukkan handshake
- [ ] Browse `https://example.com` lewat tunnel (cek IP via curl ifconfig.me)
- [ ] Disconnect WiFi 30s → reconnect → tunnel auto-recover (PersistentKeepalive)
- [ ] Logout user → login lagi → tunnel masih up (LaunchDaemon, bukan LaunchAgent)
- [ ] Reboot Mac → tunnel auto-start (`RunAtLoad`)
- [ ] Uninstall via app → service unloaded, plist dihapus, binary dibiarkan (atau dihapus)

## 8.12 Common issues

| Issue | Cause | Fix |
|---|---|---|
| `wg-quick: command not found` | Binary tidak terinstall / PATH | Verify `/usr/local/bin/wg-quick` exists, set `PATH` di plist |
| `Operation not permitted` saat start tunnel | LaunchDaemon tidak punya privilege | Pastikan plist owner `root:wheel`, mode 644 |
| Tunnel up tapi tidak ada handshake | Firewall block UDP 51820 | Test `nc -uvz n1.ndr24.com 51820` dari Mac |
| App freeze saat osascript | Code sign tidak valid | Verify `codesign --verify --deep` |
| Notarization gagal "hardened runtime missing" | Lupa flag `--options runtime` | Re-sign dengan flag |

---

[← Bab 7 Keamanan]({{ site.baseurl }}{% link docs/07-keamanan.md %}){: .btn }
[Bab 9 — CLI Reference →]({{ site.baseurl }}{% link docs/09-cli-reference.md %}){: .btn .btn-primary }
