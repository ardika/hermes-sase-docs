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
| Realtime | ⚠️ Ada tapi **tidak aktif** | semua tabel `Realtime Enabled = ❌` di production |
| RLS (Row Level Security) | ✅ Aktif di `user_data` | policy `uid() = uid` |
| Storage | ✅ Ada | tidak relevan untuk SASE |
| **Edge Functions** | ❌ **TIDAK ADA** | self-hosted tidak include Deno runtime |

**Implikasi arsitektur:**
- Client query `user_data` **langsung via PostgREST** (HTTPS + JWT, RLS-protected).
- Tidak ada Edge Function.
- Realtime subscription **tidak available** → pakai **polling tiap 5 menit** untuk detect config refresh.

> **PENTING — JANGAN UBAH SUPABASE.** Schema, RLS policy, trigger, dan kolom yang ada di `user_data` adalah **read-only** untuk implementasi ini. Dokumentasi ini diperbarui agar sejalan dengan struktur yang **sudah ada** di production. Tidak ada migration / `ALTER TABLE` / `CREATE POLICY` yang perlu dijalankan.

## 3.2 Schema `user_data` di production (TIDAK BOLEH DIUBAH)

Tabel `public.user_data` punya **23 kolom**. Yang relevan untuk implementasi SASE:

| Kolom | Type | Catatan |
|---|---|---|
| `id` | `int4` | Primary key auto-increment. Tidak dipakai di RLS. |
| `uid` | `uuid` | **Foreign key ke auth user.** Dipakai RLS untuk filter row user. |
| `configuration` | `text` | **Full WireGuard INI config string** — `[Interface]` + `[Peer]`, sudah termasuk `PrivateKey`, `Address`, `DNS`, `PublicKey`, `Endpoint`, `AllowedIPs`. **Tidak ada `PresharedKey`, `MTU`, atau `PersistentKeepalive`** di production. |
| `assigned_ip` | `text` | IP yang di-assign user di tunnel. Sudah termasuk di `Address` field di `configuration`, jadi biasanya tidak dipakai sendiri. |
| `sase_slice_id` | `int4` | Pointer ke slice/network di gateway. Diset oleh admin. |
| `user_peer_id` | `int4` | Peer ID di control plane gateway. Default 0 di sample data. |
| `sase_version` | `text` | Versi SASE. Empty string di sample data — gunakan sebagai informational, bukan dependency. |

Kolom-kolom Microsoft 365 / Azure AD (`microsoft_uid`, `application_client_id`, `client_secret_value`, `directory_tenant_id`, `email`, `microsoft_graph_api_endpoint`, `ms_ad_id`, `tfa_id`), profile user (`first_name`, `last_name`, `isadmin`, `description`), dan `organization_id`, `create_time`, `update_time`, `type` — **tidak dipakai oleh kode SASE**, tapi tetap ada di tabel.

### 3.2.1 RLS policy yang aktif di `user_data`

Sebuah policy `"Allow all usage"` apply ke semua command (`SELECT`, `INSERT`, `UPDATE`, `DELETE`):

```sql
USING (uid() = uid)
WITH CHECK (uid() = uid)
```

`uid()` adalah Supabase auth helper yang return `auth.users.id` dari JWT. Implikasinya:
- User yang login hanya bisa akses **row dengan `uid` matching user-nya sendiri**.
- Client tidak perlu pass `?id=eq.X` filter — RLS sudah enforce otomatis.
- Query simple: `GET /rest/v1/user_data?select=configuration&limit=1` akan otomatis return baris user yang login.

### 3.2.2 Yang TIDAK ada di production (jangan asumsikan)

| Asumsi sebelumnya | Realitas |
|---|---|
| Kolom `wg_config`, `wg_public_key`, `wg_status`, `wg_last_handshake` | ❌ TIDAK ada. Pakai `configuration` (full INI) saja. |
| Kolom `sase_role` | ❌ TIDAK ada. Tidak ada role-based policy at user_data level. |
| Trigger `protect_wg_config` | ❌ TIDAK ada. Client TIDAK boleh `UPDATE configuration` — disiplin di kode, bukan enforced di DB. |
| Tabel `sase_peer`, `sase_audit_log` | ❌ TIDAK ada. Audit log dilakukan di sisi client (app log) atau external. |
| `PresharedKey` di config | ❌ TIDAK dipakai. |
| `MTU`, `PersistentKeepalive` di config | ❌ TIDAK ada di config production. |
| Realtime subscription | ❌ Tidak aktif. Pakai polling. |

## 3.3 WireGuard — Embedded, BUKAN Bundle Installer

> **PENTING:** Hermes Network 360 Guard menggunakan **WireGuard yang sudah embedded di dalam project**, bukan menginstall WireGuard official client di endpoint user. Implementasi yang sudah berjalan ini **tetap dipertahankan** dalam refactor.

### 3.3.1 Windows — `tunnel.dll` via P/Invoke

WireGuard tunnel service di Windows di-load lewat P/Invoke ke `tunnel.dll` yang sudah di-bundle di root project Hermes Guard. Implementasi existing ada di folder `HermesNetwork/TunnelDll/`:

| File | Fungsi |
|---|---|
| `HermesNetwork/TunnelDll/Driver.cs` | P/Invoke ke `wireguard.dll` (`WireGuardCreateAdapter`, `WireGuardOpenAdapter`, `WireGuardGetConfiguration`, dst.) untuk adapter management |
| `HermesNetwork/TunnelDll/Service.cs` | P/Invoke ke `tunnel.dll` (`WireGuardTunnelService`) untuk tunnel runtime; plus `Service.Add(configFile, ephemeral)` yang register Windows Service |
| `HermesNetwork/TunnelDll/Win32.cs` | Wrapper SCM API (`OpenSCManager`, `CreateService`, dst.) |
| `HermesNetwork/TunnelDll/RingLogger.cs` | Log internal WireGuard |
| `tunnel.dll` di root project (~12 MB) | Embedded WireGuard tunnel binary |

**Pattern existing yang tetap dipakai:**
- Tidak ada `wireguard.exe` external. Tidak ada `C:\Program Files\WireGuard\`.
- Service Windows yang di-register adalah `WireGuardTunnel$<TunnelName>` dengan `binPath` mengarah ke binary Hermes (`HermesHelperSvc.exe` atau `HermesNetwork360Guard.exe`) plus argumen `/service <config>` — binary tersebut self-host tunnel via `Service.Run(configFile)` yang call `WireGuardTunnelService(configFile)` di `tunnel.dll`.
- Lifecycle service via `Win32.OpenSCManager` + `CreateService` / `StartService` / `ControlService` / `DeleteService` — sudah ada di `Service.cs`.

> Refactor SASE **tidak menyentuh native binding**. Helper Service hanya wrap pattern existing dengan kontrak JSON-RPC yang lebih bersih.

### 3.3.2 macOS — `wireguard-go` Bundled di App Bundle

Binary `wireguard-go` di-bundle di `.app` Hermes Guard:

```
HermesNetwork360Guard.app/
└── Contents/
    └── MacOS/
        ├── arm/wireguard-go        ← ARM64 (M1/M2/M3)
        └── intel/wireguard-go      ← x86_64 (Mac Intel)
```

Pattern existing (lihat `HermesNetwork/ViewModels/ConfigViewModel.cs`):

```csharp
// Up tunnel — pakai wg-quick yang sudah ada di /usr/local/bin/
"/usr/local/bin/bash /usr/local/bin/wg-quick up <conf-path>"

// Down tunnel
"/usr/local/bin/bash /usr/local/bin/wg-quick down <conf-path>"

// Status
"sudo /usr/local/bin/wg show"
```

> Refactor mempertahankan pattern ini — Helper Service akan call `wg-quick up/down` yang sama, hanya melalui IPC dari UI bukan langsung dari ViewModel.

### 3.3.3 Yang Berubah & Tidak Berubah

| Aspek | Sebelum (saat ini) | Setelah refactor |
|---|---|---|
| WireGuard di Windows | `tunnel.dll` P/Invoke embedded | **Sama** — `tunnel.dll` P/Invoke embedded |
| WireGuard di macOS | `wireguard-go` bundled di `.app` + `wg-quick` system | **Sama** — bundled binary tetap dipakai |
| Lifecycle tunnel | UI ↔ `IpcComService` ↔ ServiceEngine.exe ↔ TunnelDll | UI ↔ JSON-RPC ↔ HermesHelperSvc ↔ TunnelDll (existing) |
| Bundle WireGuard external installer | TIDAK (sudah embedded) | **TIDAK** (tetap embedded) |
| Install WireGuard official client | TIDAK (tidak diperlukan) | **TIDAK** (tetap tidak diperlukan) |

## 3.4 Hermes Helper Service

Helper Service adalah komponen terpisah dari `HermesNetwork360Guard.exe`. Diinstall via:

| Platform | Mekanisme | Lokasi binary |
|---|---|---|
| Windows | Installer `.msi` register Windows Service | `C:\Program Files\Hermes Network\HermesHelperSvc.exe` |
| macOS | Installer `.pkg` drop binary + LaunchDaemon plist | `/usr/local/bin/HermesHelperSvc` |

**Privilege saat install:** installer minta UAC / admin password sekali. Setelah itu service jalan persistent sebagai SYSTEM/root.

### 3.4.1 Installer responsibility

Selama install Hermes 360 Guard:

1. Install `HermesNetwork360Guard.exe` ke `C:\Program Files\Hermes Network\` (atau `/Applications/` di Mac)
2. Install `HermesHelperSvc.exe` di lokasi yang sama
3. Pastikan `tunnel.dll` (Windows, di samping binary Hermes) atau `wireguard-go` (Mac, di `Contents/MacOS/{arm,intel}/`) ter-deploy bersama app — sudah di-bundle sebagai bagian dari project Hermes, bukan installer terpisah
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

### 3.4.2 Path konvensi

| Resource | Windows | macOS |
|---|---|---|
| Helper binary | `C:\Program Files\Hermes Network\HermesHelperSvc.exe` | `/usr/local/bin/HermesHelperSvc` |
| Named-pipe | `\\.\pipe\HermesHelper` | (Mac unix-socket) |
| Unix-socket | N/A | `/var/run/hermes-helper.sock` |
| Helper log | Event Viewer (Source: HermesHelperSvc) | `os_log` |
| Hermes Guard config | `%LOCALAPPDATA%\HermesNetwork360Guard\` | `~/Library/Application Support/HermesNetwork360Guard/` |

## 3.5 Development environment

### 3.5.1 .NET tooling

| Tool | Versi | Catatan |
|---|---|---|
| .NET SDK | 8.0.x | Untuk UI app + Helper Service |
| Avalonia | 11.x | Sudah ada di project UI |
| JetBrains Rider | 2024.x | Atau Visual Studio 2022 17.8+ |

### 3.5.2 Project structure baru

```
HermesNetwork360-Avalonia.sln
├── HermesNetwork/                        ← UI app (existing)
├── HermesUpdater/                         ← existing
└── HermesHelperSvc/                       ← BARU
```

`HermesHelperSvc.csproj` minimal:

```xml
<Project Sdk="Microsoft.NET.Sdk.Worker">
  <PropertyGroup>
    <OutputType>Exe</OutputType>
    <TargetFramework>net8.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <PublishSingleFile>true</PublishSingleFile>
    <SelfContained>true</SelfContained>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.Extensions.Hosting" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting.WindowsServices" Version="8.0.0" />
    <PackageReference Include="Microsoft.Extensions.Hosting.Systemd" Version="8.0.0" />
  </ItemGroup>
</Project>
```

### 3.5.3 NuGet untuk UI app

Tambahkan ke `HermesNetwork/HermesNetwork.csproj`:

```xml
<ItemGroup>
  <PackageReference Include="System.Net.Http.Json" Version="8.0.0" />
  <PackageReference Include="Microsoft.Extensions.Logging.Abstractions" Version="8.0.0" />
  <!-- Untuk Supabase (kalau belum ada) -->
  <PackageReference Include="supabase-csharp" Version="0.16.2" />
</ItemGroup>
```

> Tidak butuh NuGet untuk crypto, ECDH, AES-GCM, named-pipe, atau key generation lokal — semua sudah di stdlib. **Plus, kita tidak generate keypair lokal sama sekali** (lihat Bab 5) — admin yang manage keypair, sudah pre-populate di `configuration`.

## 3.6 Akses & kredensial

- [ ] Repo `Hermes-Network-Inc/HermesNetwork360Guard` (write access)
- [ ] Akses Supabase self-hosted untuk **read-only inspection** (misal lihat schema, sample row test user)
- [ ] Mesin Mac dengan Apple Developer ID (untuk signing)
- [ ] User test di Supabase yang sudah punya `user_data.configuration` ter-populate

> Anda **tidak butuh akses admin Supabase**. Implementasi ini hanya membaca `user_data` lewat PostgREST dengan JWT user biasa. Tidak ada DB migration atau schema change.

## 3.7 Environment variables

### 3.7.1 UI app

| Variable | Contoh | Catatan |
|---|---|---|
| `HNGUARD_SUPABASE_URL` | `https://spdb.prod.hermesnetwork.cloud` | self-hosted URL |
| `HNGUARD_SUPABASE_ANON_KEY` | `eyJ...` | anon key, dilindungi RLS |
| `HNGUARD_SASE_TUNNEL_NAME` | `Hermes` | nama tunnel di OS service |

### 3.7.2 Helper Service

Helper Service **tidak** butuh env var network/Supabase — dia stateless dan hanya terima request dari UI lokal.

## 3.8 Verifikasi cepat

### 3.8.1 Test PostgREST + RLS

```bash
# Login user test
JWT=$(curl -X POST "$SUPABASE_URL/auth/v1/token?grant_type=password" \
  -H "apikey: $ANON" -H "Content-Type: application/json" \
  -d '{"email":"test@hermes","password":"..."}' | jq -r .access_token)

# Read user_data — RLS otomatis filter ke row user yang login
curl "$SUPABASE_URL/rest/v1/user_data?select=configuration,assigned_ip,sase_slice_id&limit=1" \
  -H "apikey: $ANON" -H "Authorization: Bearer $JWT" | jq
# Expected: 1 row dengan configuration berisi WG INI string
```

### 3.8.2 Test WireGuard manual

Pakai `configuration` dari user test:

```bash
# Save config ke /tmp
echo "$WG_CONFIG" > /tmp/test.conf

# Bring up
sudo wg-quick up /tmp/test.conf

# Verify
wg show
ping <internal-IP>

# Tear down
sudo wg-quick down /tmp/test.conf
```

### 3.8.3 Test build Helper Service skeleton

```powershell
cd HermesHelperSvc
dotnet build
```

## 3.9 Checklist sebelum lanjut

- [ ] Supabase self-hosted reachable, bisa login user test, RLS aktif
- [ ] User test punya row di `user_data` dengan `configuration` ter-populate (panjang ≥ 200 byte, mengandung `[Interface]` & `[Peer]`)
- [ ] WireGuard installed di mesin development (Win + Mac)
- [ ] Manual `wg-quick up` dengan `configuration` dari user test berhasil handshake
- [ ] Project `HermesHelperSvc` ter-create dan build
- [ ] Apple Developer ID siap (untuk Mac signing)
- [ ] Konfirmasi: **TIDAK ADA migration / ALTER TABLE / CREATE POLICY yang akan dijalankan ke Supabase**

---

[← Bab 2 Arsitektur]({{ site.baseurl }}{% link docs/02-arsitektur.md %}){: .btn }
[Bab 4 — Helper Service →]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}){: .btn .btn-primary }
