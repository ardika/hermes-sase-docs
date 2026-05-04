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

## 3.1 Supabase self-hosted

Hermes Network menggunakan **Supabase self-hosted** untuk auth + database. Implikasi yang relevan untuk dokumen ini:

| Fitur Supabase | Self-hosted | Catatan |
|---|---|---|
| Auth (GoTrue) | ✅ Ada | login user + JWT |
| PostgREST | ✅ Ada | direct REST ke Postgres |
| Realtime | ✅ Ada | subscription event row changes |
| RLS (Row Level Security) | ✅ Ada (Postgres native) | wajib enabled untuk `user_data` |
| Storage | ✅ Ada | tidak relevan untuk SASE |
| **Edge Functions** | ❌ **TIDAK ADA** | self-hosted tidak include Deno runtime |

**Implikasi arsitektur:**
- Client query `user_data` **langsung via PostgREST** (HTTPS + JWT, RLS-protected)
- Tidak ada Edge Function untuk proxy / business logic. Kalau perlu logic server-side, deploy REST service terpisah (mis. FastAPI di server Supabase yang sama).
- Realtime subscription opsional untuk push update — kalau tidak available di deployment, fallback ke polling.

### 3.1.1 Verifikasi PostgREST + RLS

Test akses PostgREST dari mesin development:

```bash
# Login user untuk dapat JWT
curl -X POST "$SUPABASE_URL/auth/v1/token?grant_type=password" \
  -H "apikey: $ANON_KEY" \
  -H "Content-Type: application/json" \
  -d '{"email":"test@hermes","password":"..."}' | jq -r .access_token

# Query user_data
curl "$SUPABASE_URL/rest/v1/user_data?select=wg_config" \
  -H "apikey: $ANON_KEY" \
  -H "Authorization: Bearer $JWT"
# Expected: array dengan satu row (RLS filter ke user yang login)
```

### 3.1.2 Schema `user_data`

Pastikan tabel sudah punya kolom yang diperlukan:

```sql
-- Cek kolom existing
\d user_data;
```

Kolom yang dipakai (sesuaikan dengan schema actual):

| Kolom | Type | Tujuan |
|---|---|---|
| `id` | UUID PK (FK ke `auth.users`) | identifier user |
| `wg_config` | TEXT | full INI WireGuard, atau JSON dengan field structured |
| `wg_public_key` | TEXT (opsional) | tempat client write-back pubkey untuk admin register |
| `wg_status` | TEXT (opsional) | last reported status (connected/disconnected/error) |
| `wg_last_handshake` | TIMESTAMPTZ (opsional) | client report periodic |

Kalau `wg_config` belum ada, tambahkan via migration:

```sql
ALTER TABLE user_data
  ADD COLUMN IF NOT EXISTS wg_config TEXT,
  ADD COLUMN IF NOT EXISTS wg_public_key TEXT,
  ADD COLUMN IF NOT EXISTS wg_last_handshake TIMESTAMPTZ;

-- RLS: user hanya bisa baca/update row sendiri
ALTER TABLE user_data ENABLE ROW LEVEL SECURITY;

CREATE POLICY "users_select_own_data"
  ON user_data FOR SELECT USING (auth.uid() = id);

CREATE POLICY "users_update_own_data"
  ON user_data FOR UPDATE USING (auth.uid() = id)
  WITH CHECK (auth.uid() = id);
```

> **Penting:** kolom yang **boleh** di-update oleh client = `wg_public_key`, `wg_last_handshake`. Kolom `wg_config` di-populate oleh ops/admin (proses di luar scope dokumen ini), client read-only.

Untuk enforce update column-level, gunakan trigger atau column-specific policy:

```sql
-- Block update kalau user coba ubah wg_config (read-only buat client)
CREATE OR REPLACE FUNCTION protect_wg_config()
RETURNS TRIGGER AS $$
BEGIN
  IF NEW.wg_config IS DISTINCT FROM OLD.wg_config THEN
    RAISE EXCEPTION 'wg_config is read-only for users';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_protect_wg_config
  BEFORE UPDATE ON user_data FOR EACH ROW
  EXECUTE FUNCTION protect_wg_config();
```

## 3.2 WireGuard di endpoint

### 3.2.1 Windows

WireGuard for Windows resmi: [wireguard.com/install](https://www.wireguard.com/install/). Yang dipakai:

| File | Lokasi default |
|---|---|
| `wireguard.exe` | `C:\Program Files\WireGuard\` |
| `wg.exe` | `C:\Program Files\WireGuard\` |
| Tunnel service | `WireGuardTunnel$<TunnelName>` |
| Config | `C:\Program Files\WireGuard\Data\Configurations\<TunnelName>.conf.dpapi.aes` |

Bundle WireGuard installer di `.msi` Hermes 360 Guard atau download saat first run.

### 3.2.2 macOS

Gunakan `wireguard-go + wg-quick` (CLI), bukan WireGuard.app dari App Store:

```bash
# Install dari Homebrew (dev) atau bundle di .pkg Hermes (prod)
brew install wireguard-tools

# Verifikasi
which wg-quick     # /usr/local/bin/wg-quick
wg --version
```

Detail signing + bundle binary di [Bab 8 — macOS]({{ site.baseurl }}{% link docs/08-mac-support.md %}).

## 3.3 Hermes Helper Service

Helper Service adalah komponen terpisah dari `HermesNetwork360Guard.exe`. Diinstall via:

| Platform | Mekanisme | Lokasi binary |
|---|---|---|
| Windows | Installer `.msi` register Windows Service | `C:\Program Files\Hermes Network\HermesHelperSvc.exe` |
| macOS | Installer `.pkg` drop binary + LaunchDaemon plist | `/usr/local/bin/HermesHelperSvc` |

**Privilege saat install:** installer minta UAC / admin password sekali. Setelah itu service jalan persistent sebagai SYSTEM/root, tidak butuh user interaction lagi.

### 3.3.1 Installer responsibility

Selama install Hermes 360 Guard:

1. Install `HermesNetwork360Guard.exe` ke `C:\Program Files\Hermes Network\` (atau `/Applications/` di Mac)
2. Install `HermesHelperSvc.exe` di lokasi yang sama
3. Bundle WireGuard binaries (`wireguard.exe`, `wg.exe`, atau `wg`/`wg-quick` di Mac)
4. Register Helper sebagai service:

   **Windows:**
   ```powershell
   sc create HermesHelperSvc `
     binPath= "C:\Program Files\Hermes Network\HermesHelperSvc.exe" `
     start= auto `
     obj= LocalSystem
   sc start HermesHelperSvc
   ```

   **macOS:**
   ```bash
   sudo cp com.hermesnetwork.helper.plist /Library/LaunchDaemons/
   sudo chown root:wheel /Library/LaunchDaemons/com.hermesnetwork.helper.plist
   sudo chmod 644 /Library/LaunchDaemons/com.hermesnetwork.helper.plist
   sudo launchctl bootstrap system /Library/LaunchDaemons/com.hermesnetwork.helper.plist
   ```

5. Helper otomatis listen di named-pipe / unix-socket — UI bisa connect tanpa elevation tambahan

### 3.3.2 Path konvensi

| Resource | Windows | macOS |
|---|---|---|
| Helper binary | `C:\Program Files\Hermes Network\HermesHelperSvc.exe` | `/usr/local/bin/HermesHelperSvc` |
| Named-pipe | `\\.\pipe\HermesHelper` | (Mac unix-socket) |
| Unix-socket | N/A | `/var/run/hermes-helper.sock` |
| Helper log | Event Viewer (Source: HermesHelperSvc) | `os_log` (subsystem `com.hermesnetwork.helper`) |
| Hermes Guard config | `%LOCALAPPDATA%\HermesNetwork360Guard\` | `~/Library/Application Support/HermesNetwork360Guard/` |

## 3.4 Development environment

### 3.4.1 .NET tooling

| Tool | Versi | Catatan |
|---|---|---|
| .NET SDK | 8.0.x | Untuk UI app + Helper Service |
| Avalonia | 11.x | Sudah ada di project UI |
| JetBrains Rider | 2024.x | Atau Visual Studio 2022 17.8+ |

### 3.4.2 Project structure baru

Tambah project baru `HermesHelperSvc` di solution:

```
HermesNetwork360-Avalonia.sln
├── HermesNetwork/                        ← UI app (existing)
├── HermesUpdater/                         ← existing
└── HermesHelperSvc/                       ← BARU
    ├── HermesHelperSvc.csproj
    └── ...
```

`HermesHelperSvc.csproj` minimal:

```xml
<Project Sdk="Microsoft.NET.Sdk.Worker">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <RootNamespace>HermesHelperSvc</RootNamespace>
    <PublishSingleFile>true</PublishSingleFile>
    <SelfContained>true</SelfContained>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting.WindowsServices" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting.Systemd" Version="8.0.0" />
    <PackageReference Include="System.IO.Pipes.AccessControl" Version="5.0.0" />
  </ItemGroup>
</Project>
```

### 3.4.3 NuGet untuk UI app

Tambah ke `HermesNetwork/HermesNetwork.csproj`:

```xml
<ItemGroup>
  <PackageReference Include="System.Net.Http.Json" Version="8.0.0" />
  <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="8.0.0" />
  <!-- Untuk Supabase (kalau belum ada) -->
  <PackageReference Include="supabase-csharp" Version="0.16.2" />
</ItemGroup>
```

> Crypto, ECDH, AES-GCM, named-pipe semuanya stdlib — tidak perlu NuGet tambahan.

## 3.5 Akses & kredensial

- [ ] Repo `Hermes-Network-Inc/HermesNetwork360Guard` (write access)
- [ ] Supabase self-hosted admin (untuk migration `user_data` schema kalau perlu)
- [ ] Mesin Mac dengan Apple Developer ID (untuk signing Helper LaunchDaemon)
- [ ] User test di Supabase yang sudah punya `user_data.wg_config` ter-populate

## 3.6 Environment variables

### 3.6.1 UI app (saat startup)

| Variable | Contoh | Catatan |
|---|---|---|
| `HNGUARD_SUPABASE_URL` | `https://supabase.hermesnetwork.cloud` | self-hosted URL |
| `HNGUARD_SUPABASE_ANON_KEY` | `eyJ...` | anon key, dilindungi RLS |
| `HNGUARD_SASE_TUNNEL_NAME` | `Hermes` | nama tunnel di OS service |

### 3.6.2 Helper Service

Helper Service **tidak** butuh env var network/Supabase — dia stateless dan hanya terima request dari UI lokal.

## 3.7 Verifikasi cepat

### 3.7.1 Test PostgREST + RLS

```bash
# Login user test
JWT=$(curl -X POST "$SUPABASE_URL/auth/v1/token?grant_type=password" \
  -H "apikey: $ANON" -H "Content-Type: application/json" \
  -d '{"email":"test@hermes","password":"..."}' | jq -r .access_token)

# Read user_data — RLS filter otomatis ke row user
curl "$SUPABASE_URL/rest/v1/user_data?select=id,wg_config" \
  -H "apikey: $ANON" -H "Authorization: Bearer $JWT" | jq
# Expected: 1 row (cuma row user yang login)
```

### 3.7.2 Test WireGuard manual

Pakai config dari `user_data.wg_config` user test:

```bash
# Save config ke /tmp
echo "$WG_CONFIG" > /tmp/test.conf

# Bring up
sudo wg-quick up /tmp/test.conf

# Verify
wg show
ping <internal-IP-yang-ada-di-AllowedIPs>

# Tear down
sudo wg-quick down /tmp/test.conf
rm /tmp/test.conf
```

### 3.7.3 Test build Helper Service skeleton

```powershell
cd HermesHelperSvc
dotnet build
# Expected: Build sukses
```

## 3.8 Checklist sebelum lanjut

- [ ] Supabase self-hosted reachable, PostgREST + RLS aktif
- [ ] `user_data.wg_config` ter-populate untuk user test
- [ ] WireGuard installed di mesin development (Win + Mac)
- [ ] Manual `wg-quick up` dengan config user test berhasil handshake
- [ ] Project `HermesHelperSvc` ter-create dan build
- [ ] Apple Developer ID siap (untuk Mac signing)

---

[← Bab 2 Arsitektur]({{ site.baseurl }}{% link docs/02-arsitektur.md %}){: .btn }
[Bab 4 — Helper Service →]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}){: .btn .btn-primary }
