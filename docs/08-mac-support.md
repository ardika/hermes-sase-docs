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

## 8.1 Komponen di macOS

Tiga komponen yang perlu di-install di Mac:

| Komponen | Lokasi | Run as |
|---|---|---|
| Hermes UI app | `/Applications/HermesNetwork360Guard.app` | Current user |
| Hermes Helper Service | `/usr/local/bin/HermesHelperSvc` + LaunchDaemon | `root` |
| WireGuard binaries | `/usr/local/bin/{wg,wg-quick,wireguard-go}` | (called by Helper) |

UI dan Helper di-distribute lewat satu `.pkg` installer.

## 8.2 PKG installer struktur

```
Hermes-Network-360-Guard.pkg
└── Payload/
    ├── Applications/HermesNetwork360Guard.app/
    ├── usr/local/bin/
    │   ├── HermesHelperSvc
    │   ├── wg
    │   ├── wg-quick
    │   └── wireguard-go
    └── Library/LaunchDaemons/
        └── com.hermesnetwork.helper.plist
```

`scripts/postinstall`:

```bash
#!/bin/bash
set -e

# Permission untuk Helper + WG binaries
chown root:wheel /usr/local/bin/HermesHelperSvc
chmod 755 /usr/local/bin/HermesHelperSvc
chown root:wheel /usr/local/bin/{wg,wg-quick,wireguard-go}
chmod 755 /usr/local/bin/{wg,wg-quick,wireguard-go}

# LaunchDaemon plist
chown root:wheel /Library/LaunchDaemons/com.hermesnetwork.helper.plist
chmod 644 /Library/LaunchDaemons/com.hermesnetwork.helper.plist

# Buat directory wireguard
mkdir -p /etc/wireguard
chmod 700 /etc/wireguard
chown root:wheel /etc/wireguard

# Bootstrap Helper LaunchDaemon
launchctl bootstrap system /Library/LaunchDaemons/com.hermesnetwork.helper.plist

exit 0
```

## 8.3 LaunchDaemon Hermes Helper

`/Library/LaunchDaemons/com.hermesnetwork.helper.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.hermesnetwork.helper</string>

    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/HermesHelperSvc</string>
    </array>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>StandardOutPath</key>
    <string>/var/log/hermes-helper.out.log</string>

    <key>StandardErrorPath</key>
    <string>/var/log/hermes-helper.err.log</string>

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

Manage:

```bash
# Lifecycle
sudo launchctl bootstrap system /Library/LaunchDaemons/com.hermesnetwork.helper.plist
sudo launchctl bootout    system/com.hermesnetwork.helper
sudo launchctl kickstart  -k system/com.hermesnetwork.helper

# Status
sudo launchctl list com.hermesnetwork.helper
```

## 8.4 LaunchDaemon WireGuard tunnel (per session)

Saat Helper apply config, dia create LaunchDaemon kedua untuk tunnel itu sendiri (lihat `MacBackend` di [Bab 4]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}) §4.3.4):

`/Library/LaunchDaemons/com.hermesnetwork.sase.Hermes.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.hermesnetwork.sase.Hermes</string>
    <key>ProgramArguments</key>
    <array>
        <string>/usr/local/bin/wg-quick</string>
        <string>up</string>
        <string>Hermes</string>
    </array>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><false/>
    <key>UserName</key><string>root</string>
</dict>
</plist>
```

> Helper sebenarnya tidak harus pakai LaunchDaemon untuk tunnel — bisa langsung `wg-quick up` saat di-request. Pakai LaunchDaemon kalau ingin auto-start saat reboot tanpa harus klik Connect.

## 8.5 Code signing

Sama seperti pattern umum:

```bash
APP_PATH="bin/Release/net8.0/osx-arm64/publish/HermesNetwork360Guard.app"
SIGN_ID="Developer ID Application: Hermes Network Inc. (XXXXXXXXXX)"

# Sign Helper binary terlebih dulu
codesign --force --options runtime --sign "$SIGN_ID" --timestamp \
  "publish/HermesHelperSvc"

# Sign WG binaries
for bin in wg wg-quick wireguard-go; do
  codesign --force --options runtime --sign "$SIGN_ID" --timestamp \
    "publish/$bin"
done

# Sign UI app + dylibs
find "$APP_PATH" -type f \( -name "*.dylib" -o -name "*.so" -o -perm +111 \) \
  -exec codesign --force --options runtime --sign "$SIGN_ID" --timestamp {} \;
codesign --force --options runtime --sign "$SIGN_ID" --timestamp \
  --entitlements "Resources/HermesNetwork360Guard.entitlements" \
  "$APP_PATH"
```

Entitlements (`HermesNetwork360Guard.entitlements`):

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.allow-jit</key><true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key><true/>
    <key>com.apple.security.network.client</key><true/>
    <key>com.apple.security.network.server</key><false/>
    <!-- Tidak butuh apple-events karena tidak osascript -->
</dict>
</plist>
```

> **Penting:** UI tidak perlu `osascript` privilege karena operasi privileged sudah lewat Helper (yang jalan as root via LaunchDaemon dari saat install). Ini lebih clean dan aman.

## 8.6 Notarization

```bash
# PKG complete (UI + Helper + WG bins)
pkgbuild --root publish-staging \
         --identifier "com.hermesnetwork.guard" \
         --version "8.4.0" \
         --install-location "/" \
         --scripts scripts/ \
         HermesNetwork360Guard.component.pkg

productsign --sign "Developer ID Installer: Hermes Network Inc. (XXXXXXXXXX)" \
            HermesNetwork360Guard.component.pkg \
            HermesNetwork360Guard.pkg

xcrun notarytool submit HermesNetwork360Guard.pkg \
  --keychain-profile AC_PASSWORD --wait

xcrun stapler staple HermesNetwork360Guard.pkg
```

## 8.7 IPC: unix-socket

Helper listen di `/var/run/hermes-helper.sock`. Saat startup, Helper:

1. Delete socket file existing (kalau ada dari crash sebelumnya)
2. Bind socket
3. `chmod 0666` (semua user bisa connect) atau `0660` dengan group khusus

```csharp
// Di MacBackend / JsonRpcServer setup
File.Delete("/var/run/hermes-helper.sock"); // ignore error
listener.Bind(new UnixDomainSocketEndPoint("/var/run/hermes-helper.sock"));
listener.Listen(64);

// Set permission supaya UI app (running as user) bisa connect
File.SetUnixFileMode("/var/run/hermes-helper.sock",
    UnixFileMode.UserRead | UnixFileMode.UserWrite |
    UnixFileMode.GroupRead | UnixFileMode.GroupWrite |
    UnixFileMode.OtherRead | UnixFileMode.OtherWrite);
```

Authentication caller via `getpeereid` — lihat [Bab 7]({{ site.baseurl }}{% link docs/07-keamanan.md %}) §7.5.2.

## 8.8 Privacy permissions (TCC)

Hermes UI tidak butuh:
- ❌ Full Disk Access
- ❌ Screen Recording
- ❌ Accessibility

Hermes UI butuh:
- ✅ Network access (otomatis granted untuk app signed)

Helper Service jalan as root, jadi tidak terkena TCC. Ini **alasan tambahan** kenapa pisah jadi 2 component:
- UI = sandbox-friendly, sedikit permission
- Helper = root-level, di luar TCC scope

## 8.9 Build matrix

Universal binary (Intel + ARM) untuk Mac:

```bash
# UI
dotnet publish -r osx-arm64 --self-contained -c Release -o publish/ui-arm64
dotnet publish -r osx-x64   --self-contained -c Release -o publish/ui-x64
lipo -create publish/ui-arm64/HermesNetwork360Guard \
            publish/ui-x64/HermesNetwork360Guard \
     -output publish/universal/HermesNetwork360Guard

# Helper
dotnet publish -p:HermesHelperSvc -r osx-arm64 --self-contained -c Release -o publish/helper-arm64
dotnet publish -p:HermesHelperSvc -r osx-x64   --self-contained -c Release -o publish/helper-x64
lipo -create publish/helper-arm64/HermesHelperSvc \
            publish/helper-x64/HermesHelperSvc \
     -output publish/universal/HermesHelperSvc
```

## 8.10 Testing checklist macOS

- [ ] Build `.pkg` di mesin Mac
- [ ] Install di Mac bersih, non-admin user
- [ ] Verify Helper running: `sudo launchctl list com.hermesnetwork.helper`
- [ ] Verify socket: `ls -la /var/run/hermes-helper.sock`
- [ ] Login user test di UI → klik Connect SASE
- [ ] Verify tunnel up: `wg show`
- [ ] Verify gateway IP: `curl ifconfig.me`
- [ ] Disable WiFi 30s → reconnect → tunnel auto-recover
- [ ] Reboot Mac → Helper auto-start, tunnel auto-up (kalau RunAtLoad di plist tunnel)
- [ ] Uninstall via app → Helper unloaded, plist dihapus

## 8.11 Common issues

| Issue | Cause | Fix |
|---|---|---|
| `launchctl list com.hermesnetwork.helper` exit 113 | Helper belum bootstrap | `sudo launchctl bootstrap system /Library/LaunchDaemons/com.hermesnetwork.helper.plist` |
| `Permission denied` saat connect socket dari UI | Mode socket terlalu ketat | `chmod 0666 /var/run/hermes-helper.sock` di Helper startup |
| Tunnel up tapi tidak ada handshake | Firewall block UDP outbound | Test `nc -uvz <gateway> 51820` |
| App freeze saat klik Connect | Helper crash | Cek `/var/log/hermes-helper.err.log` |
| Notarization gagal "hardened runtime" | Lupa `--options runtime` | Re-sign dengan flag |

---

[← Bab 7 Keamanan]({{ site.baseurl }}{% link docs/07-keamanan.md %}){: .btn }
[Bab 9 — CLI Reference →]({{ site.baseurl }}{% link docs/09-cli-reference.md %}){: .btn .btn-primary }
