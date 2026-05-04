---
layout: default
title: 2. Arsitektur
nav_order: 3
permalink: /docs/arsitektur/
---

# 2. Arsitektur
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 2.1 Constraint deployment

Tiga constraint yang membentuk arsitektur ini (lihat [Bab 1]({{ site.baseurl }}{% link docs/01-pendahuluan.md %}) §1.1):

1. **Supabase self-hosted** → tidak ada Edge Function. Komunikasi backend = direct PostgREST query (RLS) atau REST service standalone.
2. **UI tidak run as admin** → operasi privileged WireGuard butuh komponen helper terpisah dengan privilege SYSTEM/root.
3. **WG config per-user di `user_data`** → tidak ada gateway terpusat di kode; client baca config user, parse, apply via Helper.

## 2.2 Arsitektur saat ini (sebelum refactor)

```mermaid
flowchart TB
    UI[Avalonia UI<br/>HermesNetwork360Guard.exe<br/>user mode]
    SE[ServiceEngine.exe<br/>Windows Service / SYSTEM]
    WGS[WireGuard Tunnel Service<br/>WireGuardTunnel$Hermes]
    CONF[wg0.conf<br/>STATIS, di-install sekali]

    SB[(Supabase user_data<br/>WG config per-user)]

    UI -.JSON IPC<br/>magic strings.->|Code:Z1398V| SE
    SE -->|sc start| WGS
    WGS -->|baca| CONF
    UI -->|HTTPS auth| SB

    style UI fill:#3b82f6,stroke:#fff,color:#fff
    style SE fill:#dc2626,stroke:#fff,color:#fff
    style CONF fill:#dc2626,stroke:#fff,color:#fff
    style SB fill:#10b981,stroke:#fff,color:#fff
```

**Karakteristik:**
- IPC tanpa kontrak, magic strings
- ServiceEngine punya banyak responsibility (RMM + SASE + XDR + dll)
- Config statis di-install sekali, tidak refresh dari user_data

## 2.3 Arsitektur baru (target)

```mermaid
flowchart TB
    UI[Avalonia UI<br/>HermesNetwork360Guard.exe<br/>user mode]

    subgraph "Layer 2: Config Plane"
        SCC[SaseConfigClient]
    end

    subgraph "Layer 1: Privileged Helper"
        HC[HelperServiceClient<br/>typed JSON-RPC]
        HS[HermesHelperSvc<br/>Windows Service / LaunchDaemon<br/>SYSTEM / root]
        HC -.named pipe<br/>JSON-RPC.-> HS
    end

    subgraph "Layer 3: Orchestrator"
        CS[SaseConnectionService<br/>state machine + monitor]
    end

    SB[(Supabase user_data<br/>RLS-protected)]
    WGS_W[wireguard.exe<br/>+ Tunnel Service]
    WGS_M[wg-quick<br/>+ LaunchDaemon]
    GW[(WireGuard Gateway<br/>per-user endpoint)]

    UI --> CS
    CS --> SCC
    CS --> HC

    SCC -->|HTTPS + JWT user<br/>SELECT user_data| SB

    HS -.ServiceController.-> WGS_W
    HS -.launchctl.-> WGS_M
    WGS_W <-->|UDP encrypted| GW
    WGS_M <-->|UDP encrypted| GW

    style UI fill:#3b82f6,stroke:#fff,color:#fff
    style SCC fill:#8b5cf6,stroke:#fff,color:#fff
    style HC fill:#8b5cf6,stroke:#fff,color:#fff
    style HS fill:#a855f7,stroke:#fff,color:#fff
    style CS fill:#8b5cf6,stroke:#fff,color:#fff
    style SB fill:#10b981,stroke:#fff,color:#fff
```

**Karakteristik:**
- **Empat komponen jelas:** UI (user), Helper (SYSTEM), Supabase, WireGuard
- **IPC bersih:** named-pipe dengan typed JSON-RPC, schema versioned
- **Config dinamis:** read dari Supabase `user_data` saat login + on-demand
- **Surface area Helper minimal:** hanya verb terbatas untuk privileged ops
- **Tidak ada Edge Function:** PostgREST + RLS sudah cukup

## 2.4 Empat komponen: tanggung jawab

### Komponen A — Hermes UI (Avalonia, user mode)

Run di context user biasa. Tidak punya privilege admin.

**Yang dilakukan:**
- Login Supabase, simpan JWT di OS keychain
- Read `user_data` user lewat HTTPS + JWT
- Parse WG config dari `user_data.wg_config`
- Generate per-device WireGuard keypair (`wg genkey`), simpan local DPAPI/Keychain
- (Opsional) Write public key kembali ke `user_data` via `UPDATE`
- Request operasi privileged ke Hermes Helper via named-pipe IPC
- Tampilkan UI status (Connected/Disconnected, last handshake, throughput)

**Yang TIDAK dilakukan:**
- Tidak install/start/stop Windows service
- Tidak write config ke `C:\Program Files\WireGuard\Data\`
- Tidak shell out ke `wireguard.exe`/`wg-quick`
- Tidak pegang API key SASE / token privileged

### Komponen B — Hermes Helper Service (SYSTEM/root)

Komponen baru yang menggantikan peran SASE-relevant di `ServiceEngine.exe` lama, tapi dengan **kontrak typed dan scope minimal**.

| Platform | Bentuk | Cara install |
|---|---|---|
| Windows | `HermesHelperSvc.exe` jalan sebagai Windows Service `LocalSystem` | Installer `.msi` register service, start otomatis di reboot |
| macOS | LaunchDaemon `com.hermesnetwork.helper` jalan sebagai `root` | `.pkg` installer drop plist + binary, `launchctl bootstrap` |

**Yang dilakukan (verb terbatas):**

```typescript
// Schema kontrak JSON-RPC v1
type Verb =
  | "Ping"                                  // health check
  | "InstallTunnel" | "UninstallTunnel"
  | "ApplyConfig"
  | "StartTunnel" | "StopTunnel"
  | "GetStatus";
```

**Yang TIDAK dilakukan:**
- Tidak shell-out arbitrary command
- Tidak baca/tulis file di luar lokasi WireGuard yang resmi
- Tidak akses Supabase / network external (semua state input dari UI)
- Tidak cache state (stateless — tiap request fresh)

**Prinsip desain:**
- **Authenticate caller**: terima request hanya dari local user yang sama (Win: token check, Mac: peer credential check)
- **Whitelist input**: nama tunnel hanya regex `[A-Za-z0-9_-]{1,32}`, config divalidasi sebagai INI WireGuard valid
- **Audit log**: tiap request tercatat ke Event Viewer (Win) / `os_log` (Mac)

**Lokasi rekomendasi di repo:**
```
HermesHelperSvc/                           ← project terpisah dari HermesNetwork
├── HermesHelperSvc.csproj
├── Program.cs                             ← Windows Service / launchd entry
├── Rpc/
│   ├── JsonRpcServer.cs                   ← named-pipe / unix-socket listener
│   ├── RpcRequest.cs / RpcResponse.cs     ← contract
│   └── Handlers/
│       ├── PingHandler.cs
│       ├── TunnelInstallHandler.cs
│       ├── TunnelStartStopHandler.cs
│       └── TunnelStatusHandler.cs
├── Os/
│   ├── IOsBackend.cs
│   ├── WindowsBackend.cs                  ← wireguard.exe / sc.exe / ServiceController
│   └── MacBackend.cs                      ← wg-quick / launchctl
└── Auth/
    └── CallerAuthenticator.cs             ← validate same-user / same-machine
```

Detail implementasi di [Bab 4 — Helper Service]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}).

### Komponen C — `SaseConfigClient` (di UI app)

**Tanggung jawab:** Baca per-user config dari Supabase `user_data`.

| Yang dilakukan | Yang TIDAK dilakukan |
|---|---|
| `SELECT wg_config FROM user_data WHERE id = auth.uid()` | Generate WG keypair sendiri (itu di KeyStore) |
| Parse config string → `SaseConfigDto` | Validate gateway reachable |
| Subscribe Supabase Realtime `user_data` (opsional) | Apply config ke tunnel (itu Helper) |
| Write public key kembali via `UPDATE` | Pegang API key privileged |

**Lokasi:**
```
HermesNetwork/
└── Sase/
    └── Config/
        ├── SaseConfigClient.cs
        ├── ISaseConfigClient.cs
        ├── KeyStore.cs                     ← per-device keypair, DPAPI/Keychain
        └── Models/
            └── SaseConfigDto.cs
```

Detail di [Bab 5 — Config Service]({{ site.baseurl }}{% link docs/05-config-service.md %}).

### Komponen D — `SaseConnectionService` (orchestrator)

**Tanggung jawab:** State machine + background monitor, glue antara A/B/C.

```
HermesNetwork/Sase/SaseConnectionService.cs
```

Logika tipikal `ConnectAsync()`:
1. `SaseConfigClient.GetConfigAsync()` → dapat config terbaru dari user_data
2. (Opsional) Inject local PrivateKey dari KeyStore ke config string
3. `HelperServiceClient.ApplyConfigAsync(name, content)` → Helper write config + reload
4. `HelperServiceClient.StartTunnelAsync(name)` → Helper `sc start` / `launchctl kickstart`
5. Polling `HelperServiceClient.GetStatusAsync(name)` sampai handshake fresh
6. Spawn background monitor task

Detail di [Bab 6 — Connection Flow]({{ site.baseurl }}{% link docs/06-connection-flow.md %}).

## 2.5 Aliran data: "user klik Connect SASE"

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant UI as Avalonia UI<br/>(user)
    participant CS as ConnectionService
    participant SCC as SaseConfigClient
    participant SB as Supabase<br/>(PostgREST + RLS)
    participant HC as HelperClient
    participant HS as Hermes Helper<br/>(SYSTEM)
    participant WG as WireGuard service
    participant GW as Gateway (dari config)

    U->>UI: Klik "Connect SASE"
    UI->>CS: ConnectAsync()

    CS->>SCC: GetConfigAsync()
    SCC->>SB: SELECT wg_config FROM user_data<br/>WHERE id = auth.uid()
    SB-->>SCC: { wg_config: "[Interface]\\n..." }
    SCC->>SCC: Parse + inject local PrivateKey<br/>(dari KeyStore)
    SCC-->>CS: SaseConfigDto

    CS->>HC: ApplyConfig(name, content)
    HC->>HS: { jsonrpc: "2.0", method: "ApplyConfig", ... }
    HS->>HS: Validate input
    HS->>HS: Write file ke C:\Program Files\WireGuard\Data\
    HS-->>HC: { result: "ok" }

    CS->>HC: StartTunnel(name)
    HC->>HS: { method: "StartTunnel", ... }
    HS->>WG: sc start WireGuardTunnel$Hermes
    WG->>GW: Handshake UDP
    GW-->>WG: Handshake OK
    WG-->>HS: Service running
    HS-->>HC: { result: "ok" }

    CS->>HC: GetStatus(name) (polling 1s)
    HC->>HS: { method: "GetStatus", ... }
    HS->>HS: wg show <name> dump
    HS-->>HC: { lastHandshake: <ts>, ... }
    HC-->>CS: TunnelStatus

    CS-->>UI: Connected
    UI-->>U: ✅ "Connected" indicator
```

**Catatan:**
- **Private key WG TIDAK pernah meninggalkan client device.** Disimpan di DPAPI/Keychain.
- **API key SASE TIDAK ada** — gateway peer config sudah tersimpan di user_data per user, di-populate oleh ops/admin via tooling lain.
- Helper hanya tahu "apply config X, start tunnel Y" — tidak tahu siapa user atau apa policy.

## 2.6 Aliran data: "config refresh otomatis"

```mermaid
sequenceDiagram
    autonumber
    participant SB as Supabase Realtime
    participant CS as ConnectionService
    participant SCC as SaseConfigClient
    participant HC as HelperClient
    participant HS as Hermes Helper

    Note over CS: User connected. Background subscribe<br/>ke user_data updates via Realtime
    SB-->>CS: { event: "UPDATE", row: { wg_config: "..." } }

    CS->>SCC: GetConfigAsync()
    SCC-->>CS: New SaseConfigDto

    CS->>HC: ApplyConfig(name, newContent)
    HC->>HS: ApplyConfig
    HS->>HS: Replace file + restart WG service
    HS-->>HC: ok
    Note over HS: WireGuard re-handshake otomatis<br/>(tetap connected, tidak drop UX)
```

Untuk fallback (kalau Realtime tidak available di self-hosted Supabase): polling tiap 5 menit query `user_data` untuk cek `updated_at` change.

## 2.7 Trade-off & rationale

| Keputusan | Alternatif | Alasan |
|---|---|---|
| Hermes Helper Service terpisah | UAC dialog tiap operasi | UAC tiap klik = UX rusak. Helper sekali install, jalan persistent. |
| Direct PostgREST query (no Edge Function) | Bikin REST service standalone | Self-hosted Supabase support PostgREST + RLS lengkap. Tidak perlu deploy lebih banyak. |
| Named-pipe + JSON-RPC | gRPC, REST-over-HTTP, COM | Named-pipe = built-in OS authentication, ringan, sudah pattern di Windows. JSON-RPC = simple kontrak. |
| Per-device keypair di-generate client | Keypair dari user_data | Private key tidak pernah leave device. Client write public key kembali via UPDATE. |
| Stateless Helper | Helper cache state | Stateless = mudah di-audit, lebih sedikit bug. |
| Verb whitelist Helper | Generic "RunCommand" | Generic = backdoor risk. Whitelist = audit trail jelas. |
| Tetap WireGuard | OpenZiti / Tailscale / NetBird | WireGuard sudah running, tested, fast. |

## 2.8 Apa yang TIDAK berubah

- ✅ WireGuard data plane: binary `wireguard.exe` / `wg-quick` resmi
- ✅ Crypto WireGuard
- ✅ Supabase `user_data` schema (kita hanya read/optional update kolom yang sudah ada)
- ✅ Tab UI Avalonia struktur

## 2.9 Apa yang berubah

- ❌ `ServiceEngine.exe` SASE-related — DIHAPUS, diganti `HermesHelperSvc`
- ❌ Custom IPC `Code: "Z1398V"` magic strings — DIHAPUS, diganti typed JSON-RPC
- ➕ `HermesHelperSvc/` — project baru untuk Helper Service
- ➕ `HermesNetwork/Sase/Helper/` — client side untuk panggil Helper
- ➕ `HermesNetwork/Sase/Config/` — Supabase user_data reader
- ➕ Per-device WG keypair (DPAPI/Keychain)
- 🔄 Lifecycle WireGuard: lewat Helper IPC instead of `IpcComService`

---

[← Bab 1 Pendahuluan]({{ site.baseurl }}{% link docs/01-pendahuluan.md %}){: .btn }
[Bab 3 — Prasyarat →]({{ site.baseurl }}{% link docs/03-prasyarat.md %}){: .btn .btn-primary }
