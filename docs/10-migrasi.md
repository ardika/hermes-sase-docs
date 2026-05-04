---
layout: default
title: 10. Rencana Migrasi
nav_order: 11
permalink: /docs/migrasi/
---

# 10. Rencana Migrasi
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 10.1 Filosofi: paralel, bukan big-bang

Sama dengan refactor TRMM. Arsitektur baru ditambahkan **berdampingan** dengan kode lama, di-control feature flag. Setelah stabil di produksi, kode lama dihapus.

Manfaat:
- Rollback gampang (toggle flag)
- VPN user tidak interrupted selama migrasi
- Bisa bertahap per-OS
- Tim paralel: backend (schema migration) + frontend (helper service + UI integration)

## 10.2 Phase overview

| Phase | Durasi | Aktivitas | Hasil |
|---|---|---|---|
| **0** | 1 minggu | Audit code SASE existing + schema `user_data` | Mapping IPC → tujuan baru, schema disetujui |
| **1** | 1 minggu | Schema migration `user_data` + RLS policy | DB ready di staging |
| **2** | 2 minggu | Implementasi `HermesHelperSvc` (Win + Mac) | Build + install standalone, ter-test |
| **3** | 1 minggu | Implementasi `HelperServiceClient` + `SaseConfigClient` | Standalone test, belum integrate UI |
| **4** | 1 minggu | `SaseConnectionService` + UI binding | Tab SASE dengan flag `UseNewSaseStack=true` |
| **5** | 2 minggu | Beta + 100% rollout | Production users di arsitektur baru |
| **6** | 1 minggu | Cleanup IpcComService SASE-related | Surface area bersih |
| **Total** | **~9 minggu** | | Production fully migrated |

## 10.3 Phase 0 — Audit

### Tugas

1. Search occurrence di repo:

   ```powershell
   cd F:\Upwork\CharlesUSA\HermesNetwork360-Avalonia
   git grep -nE "WireGuard|wireguard|SASE|sase" -- '*.cs' > audit-sase.txt
   git grep -nE '"Code".*"SASE"' -- '*.cs' >> audit-sase.txt
   ```

2. Catat tabel:

   | File:Line | Operasi sekarang | Tujuan baru |
   |---|---|---|
   | `ConfigViewModel.cs:1555` | IPC: SASE Start | `_helperClient.StartTunnelAsync("Hermes")` via `SaseConnectionService.ConnectAsync()` |
   | `ConfigViewModel.cs:390` | sc query state | `_helperClient.GetStatusAsync("Hermes")` |
   | `... XDR Start` | (di luar scope SASE) | (lihat doc TRMM) |

3. Verifikasi schema `user_data`:

   ```sql
   \d user_data;
   -- Check apakah wg_config column sudah ada
   SELECT id, length(wg_config) FROM user_data WHERE id = '<test-user>';
   ```

   Kalau `wg_config` belum ada / belum populated, koordinasi dengan tim ops untuk:
   - Tambah kolom (lihat [Bab 5]({{ site.baseurl }}{% link docs/05-config-service.md %}) §5.2)
   - Populate config untuk test users

4. Dokumentasikan **format `wg_config`** di Supabase production: INI string atau JSON? Apakah include PrivateKey?

### Exit criteria

- [ ] Mapping IPC → tujuan baru lengkap
- [ ] Schema `user_data` ter-dokumentasi
- [ ] Format `wg_config` confirmed dengan ops
- [ ] Sign-off security tentang trust model (PrivateKey location, PSK rotation)

## 10.4 Phase 1 — Schema migration

### Tugas

1. Apply migration ke Supabase staging:

   ```sql
   ALTER TABLE user_data
     ADD COLUMN IF NOT EXISTS wg_public_key TEXT,
     ADD COLUMN IF NOT EXISTS wg_status TEXT,
     ADD COLUMN IF NOT EXISTS wg_last_handshake TIMESTAMPTZ;

   ALTER TABLE user_data ENABLE ROW LEVEL SECURITY;

   CREATE POLICY "users_select_own"
     ON user_data FOR SELECT USING (auth.uid() = id);

   CREATE POLICY "users_update_own"
     ON user_data FOR UPDATE USING (auth.uid() = id)
     WITH CHECK (auth.uid() = id);

   -- Trigger protect wg_config
   CREATE OR REPLACE FUNCTION protect_wg_config() ...;
   CREATE TRIGGER trg_protect_wg_config ...;
   ```

2. Buat tabel `sase_audit_log` (opsional tapi recommended)

3. Test akses via PostgREST dengan JWT user test

### Exit criteria

- [ ] Migration applied di staging
- [ ] Test user bisa SELECT user_data sendiri
- [ ] Test user TIDAK bisa UPDATE wg_config (trigger reject)
- [ ] Sign-off DBA

## 10.5 Phase 2 — Hermes Helper Service

### Tugas

1. Buat project `HermesHelperSvc` baru di solution
2. Implementasi `JsonRpcServer`, `IOsBackend`, `WindowsBackend`, `MacBackend`, `CallerAuthenticator` per [Bab 4]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %})
3. Unit test semua handler
4. Integration test: install Helper di VM bersih, dari script test panggil verb-verb

   **Windows (test script PowerShell):**
   ```powershell
   $pipe = New-Object IO.Pipes.NamedPipeClientStream(
     ".", "HermesHelper", "InOut", "Asynchronous")
   $pipe.Connect(5000)

   $req = '{"jsonrpc":"2.0","id":"1","method":"Ping"}'
   $bytes = [Text.Encoding]::UTF8.GetBytes($req)
   $hdr = [BitConverter]::GetBytes([Net.IPAddress]::HostToNetworkOrder($bytes.Length))
   $pipe.Write($hdr, 0, 4)
   $pipe.Write($bytes, 0, $bytes.Length)
   # ... read response
   ```

5. Manual integration test: install Helper, lalu test full flow apply config + start + status

### Exit criteria

- [ ] CI build hijau (Win + Mac)
- [ ] Unit test coverage ≥ 80%
- [ ] Integration test sukses Win + Mac
- [ ] Code review approved
- [ ] Helper signed dengan Authenticode (Win) + Developer ID (Mac)

## 10.6 Phase 3 — Client side: `HelperServiceClient` + `SaseConfigClient`

### Tugas

1. Buat folder `HermesNetwork/Sase/Helper/` + `HermesNetwork/Sase/Config/`
2. Implementasi `HelperServiceClient` (named-pipe / unix-socket)
3. Implementasi `SaseConfigClient` + `KeyStore`
4. Test integration dengan staging Supabase + Helper di VM:

   ```csharp
   [Fact]
   public async Task EndToEnd_GetConfigThenApplyToHelper()
   {
       var config = await _configClient.GetConfigAsync();
       config.Should().NotBeNullOrWhitespace();
       await _helperClient.ApplyConfigAsync("Hermes", config);
       var st = await _helperClient.GetStatusAsync("Hermes");
       st.State.Should().BeOneOf("Stopped", "Up");   // tergantung apakah service ter-install
   }
   ```

### Exit criteria

- [ ] Standalone test panggil Supabase + Helper berhasil
- [ ] KeyStore round-trip Win + Mac
- [ ] PR merged

## 10.7 Phase 4 — `SaseConnectionService` + UI

### Tugas

1. Implementasi `SaseConnectionService` per [Bab 6]({{ site.baseurl }}{% link docs/06-connection-flow.md %})
2. Refactor `SaseTabViewModel` (atau tab lain yang punya tombol Connect SASE) untuk pakai `SaseConnectionService`
3. Tambah feature flag `appsettings.json`:

   ```json
   { "FeatureFlags": { "UseNewSaseStack": false } }
   ```

4. DI registration:

   ```csharp
   if (config.FeatureFlags.UseNewSaseStack)
   {
       services.AddSingleton<IHelperServiceClient, HelperServiceClient>();
       services.AddSingleton<ISaseConfigClient, SaseConfigClient>();
       services.AddSingleton<ISaseConnectionService, SaseConnectionService>();
       services.AddSingleton<KeyStore>();
   }
   else
   {
       services.AddSingleton<ISaseConnectionService, LegacySaseConnectionService>();
   }
   ```

5. Manual test toggle flag → swap behavior

### Exit criteria

- [ ] Flag `false` → behavior lama identik
- [ ] Flag `true` → flow baru jalan end-to-end di Win + Mac
- [ ] State update real-time di UI (StateChanges subscription)
- [ ] No crash saat toggle flag
- [ ] Network change handler bekerja

## 10.8 Phase 5 — Beta + 100% rollout

### Strategi

1. **Internal QA** 1 minggu — flag `true` di staging
2. **Beta** 1 minggu — 5% production users dengan flag `true`
3. **Monitor metrics**:
   - SASE connect success rate
   - Time-to-handshake
   - Disconnect rate per hour
   - Helper IPC error rate
4. **100% rollout** kalau metric stable
5. **Rollback** plan: hot-fix release dengan flag default `false`

### Telemetry

Push event ke `sase_audit_log`:

- `connect.attempt`, `connect.success`, `connect.timeout`, `connect.error`
- `disconnect.user`, `disconnect.unexpected`
- `reconnect.auto.success`, `reconnect.auto.fail`
- `helper.rpc.error.<code>`

Query Supabase periodic untuk monitor.

### Exit criteria

- [ ] 100% rollout sukses 1 minggu tanpa rollback
- [ ] Connect success rate ≥ 95%
- [ ] No user complaint signifikan

## 10.9 Phase 6 — Cleanup

### Tugas

1. Verifikasi tidak ada reference legacy:

   ```bash
   git grep "LegacySaseConnectionService"      # 0 hits
   git grep '"Code".*"SASE"' -- '*.cs'         # 0 hits
   git grep "ExecuteSaseButton" -- '*.cs'      # 0 hits
   ```

2. Hapus file:
   ```
   HermesNetwork/Service/SaseLegacyHandler.cs
   HermesNetwork/Sase.Old/                     (jika ada)
   ```

3. Hapus feature flag `UseNewSaseStack`
4. **Catatan**: ServiceEngine.exe **tetap** ada selama masih dipakai oleh komponen lain (XDR, dll). Setelah refactor TRMM + SASE selesai, ServiceEngine bisa di-evaluate untuk dihapus.

### Exit criteria

- [ ] CI hijau
- [ ] Release notes mention "SASE rewrite — Helper Service + Supabase user_data direct"
- [ ] 1 minggu produksi tanpa regression

## 10.10 Rollback strategy per phase

| Phase | Rollback |
|---|---|
| 0 | N/A (audit) |
| 1 | Drop columns yang ditambah (kalau possible) |
| 2 | Uninstall Helper Service via installer; revert PR |
| 3 | Revert PR; arsitektur lama tetap pakai IPC |
| 4 | Toggle flag false |
| 5 | Hot-fix release flag default false |
| 6 | Restore dari git history |

## 10.11 Risk register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Helper Service tidak terinstall di endpoint user | Medium | High | Bundle di installer, check Ping di startup |
| Permission issue caller authentication | Medium | Medium | Comprehensive testing multi-user scenarios |
| `wg_config` di Supabase format inkonsisten antar user | High | High | Validator + fail early dengan pesan jelas |
| Apple Developer ID expire | Low | Critical | Calendar reminder 30 hari |
| Conflict dengan VPN client lain (NordVPN, dll.) | Medium | Medium | Document incompatibility, detect saat startup |
| Kill switch terlalu agresif → user lock out | Medium | High | Default soft-fail; opt-in kill switch per role |
| Realtime subscription tidak available di self-hosted | Medium | Low | Fallback ke polling 5 menit |

## 10.12 Definition of Done

Migrasi **selesai** kalau:

- [ ] Phase 0–6 semua exit criteria tercentang
- [ ] `LegacySaseConnectionService` dihapus dari main
- [ ] Feature flag `UseNewSaseStack` dihilangkan
- [ ] Telemetry produksi 2 minggu menunjukkan no increase di error rate
- [ ] Onboarding doc tim engineering update
- [ ] Tidak ada call IPC custom yang menyentuh SASE
- [ ] Helper Service signed + notarized untuk Mac, Authenticode signed untuk Win
- [ ] Schema `user_data` migrasi applied di production

---

[← Bab 9 CLI Reference]({{ site.baseurl }}{% link docs/09-cli-reference.md %}){: .btn }
[Bab 11 — Troubleshooting →]({{ site.baseurl }}{% link docs/11-troubleshooting.md %}){: .btn .btn-primary }
