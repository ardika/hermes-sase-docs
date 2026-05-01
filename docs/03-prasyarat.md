---
layout: default
title: 3. Prasyarat
nav_order: 4
permalink: /docs/prasyarat/
---

# 3. Prasyarat
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 3.1 SASE Gateway

Anda perlu akses ke gateway SASE yang sudah berjalan. Hermes Network menggunakan:

| Komponen | URL / endpoint |
|---|---|
| SASE gateway | `n1.ndr24.com:51820/UDP` |
| SASE control plane API | (vendor-specific — minta detail dari ops) |

Kalau Anda perlu instance staging, ikuti panduan vendor SASE Anda untuk men-deploy gateway dan control plane.

### 3.1.1 Membuat API key untuk control plane

Untuk Edge Function bisa men-register peer secara otomatis, butuh API key dari control plane SASE:

1. Login ke dashboard admin gateway
2. Pergi ke **Settings → API Keys**
3. Buat key dengan scope **peer:read,write** (atau setara)
4. Salin key — disimpan di Supabase secrets, bukan di binary

> **Penting:** API key ini **hanya boleh disimpan di Supabase Edge Function**, jangan pernah embed di desktop client. Detail di [Bab 7]({{ site.baseurl }}{% link docs/07-keamanan.md %}).

### 3.1.2 Tentukan range AllowedIPs untuk tiap role

Salah satu manfaat refactor adalah **per-role AllowedIPs** (split-tunnel berbasis policy). Sebelum coding, definisikan kebijakan dengan tim ops:

| Role | AllowedIPs | Catatan |
|---|---|---|
| `engineer` | `10.0.0.0/8, 192.168.0.0/16` | Akses internal corporate network |
| `executive` | `0.0.0.0/0` | Full tunnel — semua trafik via gateway |
| `intern` | `10.10.0.0/16` | Hanya subnet tertentu |
| `contractor` | `10.20.5.0/24` | Akses minimal ke 1 server |

Mapping ini akan disimpan di Supabase atau di-hardcode di Edge Function (tergantung preferensi).

## 3.2 WireGuard di endpoint

### 3.2.1 Windows

WireGuard for Windows resmi tersedia di [wireguard.com/install](https://www.wireguard.com/install/). Yang dipakai:

| File | Lokasi default | Fungsi |
|---|---|---|
| `wireguard.exe` | `C:\Program Files\WireGuard\` | UI + CLI installer untuk tunnel service |
| `wg.exe` | `C:\Program Files\WireGuard\` | CLI untuk query status (`wg show`) |
| Tunnel service | `WireGuardTunnel$<TunnelName>` di Service Manager | Service per tunnel |
| Config | `C:\Program Files\WireGuard\Data\Configurations\<TunnelName>.conf.dpapi.aes` | Encrypted dengan DPAPI |

Installer Hermes harus **bundle WireGuard installer** atau **download saat first run**. Versi minimum: 0.5.3.

```powershell
# Verifikasi installed
Get-Command wireguard -ErrorAction SilentlyContinue
& "C:\Program Files\WireGuard\wireguard.exe" /version
```

### 3.2.2 macOS

Dua opsi:

| Opsi | Pro | Con |
|---|---|---|
| **WireGuard.app** dari Mac App Store | UI built-in, Network Extension, sandboxed | Susah otomasi, butuh user interaction |
| **wireguard-go + wg-quick** (Homebrew / pkg) | Full automation via CLI | Harus jalan sebagai LaunchDaemon, butuh signed kalau didistribusikan |

Untuk Hermes Network 360 Guard yang butuh otomasi, gunakan **wireguard-go + wg-quick** lewat install package custom:

```bash
# Test install (Homebrew, dev only)
brew install wireguard-tools

# Verifikasi
which wg
which wg-quick
wg --version
```

Untuk distribusi production, package `.pkg` install:
- `/usr/local/bin/wg`
- `/usr/local/bin/wg-quick`
- `/usr/local/bin/wireguard-go`
- LaunchDaemon plist di `/Library/LaunchDaemons/com.hermesnetwork.sase.plist`

Detail signing + notarization di [Bab 8 — Mac Support]({{ site.baseurl }}{% link docs/08-mac-support.md %}).

## 3.3 Development environment

### 3.3.1 Tooling .NET

| Tool | Versi minimum | Catatan |
|---|---|---|
| .NET SDK | 8.0 | Pakai 8.0.x latest stable |
| Avalonia | 11.x | Sudah ada di project |
| JetBrains Rider | 2024.x | Atau Visual Studio 2022 17.8+ |
| Git | 2.40+ | |

### 3.3.2 NuGet packages baru

Tambahkan ke `HermesNetwork/HermesNetwork.csproj`:

```xml
<ItemGroup>
  <!-- Layer 2: HTTP client utilities -->
  <PackageReference Include="System.Net.Http.Json" Version="8.0.0" />
  <PackageReference Include="Microsoft.Extensions.Http.Polly" Version="8.0.0" />

  <!-- Logging -->
  <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="8.0.0" />
</ItemGroup>
```

> **Tidak butuh NuGet** untuk crypto Curve25519 — kita generate keypair via `wg genkey` (CLI WireGuard) yang sudah terbundle dengan installer-nya.

### 3.3.3 Tooling backend

| Tool | Versi minimum | Catatan |
|---|---|---|
| Supabase CLI | 1.150+ | `npm i -g supabase` |
| Deno | 1.40+ | Otomatis terinstall via Supabase CLI |

```bash
supabase login
supabase link --project-ref YOUR_PROJECT_REF
```

## 3.4 Akses & kredensial

- [ ] Repo `Hermes-Network-Inc/HermesNetwork360Guard` di GitHub (write access)
- [ ] Supabase project (admin role untuk deploy edge function)
- [ ] Akun admin SASE control plane (untuk generate API key + cek peer state)
- [ ] SSH ke gateway (untuk debug peer registration kalau ada masalah)

## 3.5 Environment variables

### 3.5.1 Desktop client (di-set saat startup app)

| Variable | Contoh | Asal |
|---|---|---|
| `HNGUARD_SUPABASE_URL` | `https://xxx.supabase.co` | Supabase dashboard |
| `HNGUARD_SUPABASE_ANON_KEY` | `eyJ...` | Supabase dashboard |
| `HNGUARD_SASE_TUNNEL_NAME` | `Hermes` | Statis, sama dengan label di OS service |

### 3.5.2 Supabase Edge Function (set via `supabase secrets`)

| Variable | Contoh | Asal |
|---|---|---|
| `SASE_API_URL` | `https://api.sase-control.example.com` | Vendor-specific |
| `SASE_API_KEY` | `your-api-key-here` | step 3.1.1 |
| `SASE_GATEWAY_ENDPOINT` | `n1.ndr24.com:51820` | Static |
| `SASE_GATEWAY_PUBLIC_KEY` | `b64-public-key=` | Public key WireGuard gateway |
| `SASE_DEFAULT_DNS` | `10.0.0.1, 1.1.1.1` | DNS yang dipush ke client |

```bash
supabase secrets set SASE_API_URL=https://api.sase-control.example.com
supabase secrets set SASE_API_KEY=your-api-key-here
supabase secrets set SASE_GATEWAY_ENDPOINT=n1.ndr24.com:51820
supabase secrets set SASE_GATEWAY_PUBLIC_KEY=b64-public-key=
supabase secrets set SASE_DEFAULT_DNS=10.0.0.1,1.1.1.1
```

## 3.6 Verifikasi cepat

### 3.6.1 Test WireGuard installed di endpoint

**Windows:**
```powershell
& "C:\Program Files\WireGuard\wg.exe" --version
# Output: wireguard-tools v1.0.20210914 - https://git.zx2c4.com/wireguard-tools/
```

**macOS:**
```bash
wg --version
wg-quick --help | head -5
```

### 3.6.2 Test handshake manual ke gateway

Generate keypair throwaway, register manual via control plane, run sekali, lalu cleanup:

**Windows / macOS / Linux (sama):**
```bash
# Generate key
wg genkey | tee /tmp/priv.key | wg pubkey > /tmp/pub.key
echo "Public key: $(cat /tmp/pub.key)"

# Buat config minimal (manual register peer di gateway dashboard dulu)
cat > /tmp/test.conf <<EOF
[Interface]
PrivateKey = $(cat /tmp/priv.key)
Address = 10.99.99.99/32

[Peer]
PublicKey = <gateway-public-key>
Endpoint = n1.ndr24.com:51820
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
EOF

# Bring up
sudo wg-quick up /tmp/test.conf

# Test
wg show
ping -c 3 10.0.0.1

# Bring down + cleanup
sudo wg-quick down /tmp/test.conf
rm /tmp/test.conf /tmp/priv.key /tmp/pub.key
```

Kalau handshake berhasil, infrastruktur SASE sudah siap dipakai untuk implementasi automation.

### 3.6.3 Test deploy stub edge function

```bash
cd <your-supabase-project>
supabase functions new sase-config
supabase functions deploy sase-config
```

Verifikasi muncul di Supabase dashboard → Functions.

## 3.7 Checklist sebelum lanjut

Pastikan semua centang sebelum lanjut ke [Bab 4]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}):

- [ ] SASE API key sudah dibuat & disimpan di password manager
- [ ] Mapping role → AllowedIPs sudah disetujui ops
- [ ] WireGuard installed di mesin development (Win + Mac)
- [ ] Manual handshake test ke gateway berhasil
- [ ] Supabase CLI bisa login + link
- [ ] Edge function stub berhasil di-deploy
- [ ] Public key gateway WireGuard sudah dicatat

---

[← Bab 2 Arsitektur]({{ site.baseurl }}{% link docs/02-arsitektur.md %}){: .btn }
[Bab 4 — TunnelSupervisor →]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}){: .btn .btn-primary }
