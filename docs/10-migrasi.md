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
- Tim paralel: ops (sign-off konfirmasi schema existing) + frontend (helper service + UI integration)

## 10.2 Phase overview

> **PENTING:** Tidak ada phase schema migration. Schema Supabase **dilarang diubah** — implementasi pakai kolom existing (`uid`, `configuration`, `assigned_ip`, `sase_slice_id`, `sase_version`) dan RLS yang sudah aktif.

| Phase | Durasi | Aktivitas | Hasil |
|---|---|---|---|
| **0** | 1 minggu | Audit code SASE existing + verifikasi schema `user_data` production | Mapping IPC → tujuan baru, format `configuration` confirmed |
| **1** | 2 minggu | Implementasi `HermesHelperSvc` (Win + Mac) | Build + install standalone, ter-test |
| **2** | 1 minggu | Implementasi `HelperServiceClient` + `SaseConfigClient` | Standalone test, belum integrate UI |
| **3** | 1 minggu | `SaseConnectionService` + UI binding | Tab SASE dengan flag `UseNewSaseStack=true` |
| **4** | 2 minggu | Beta + 100% rollout | Production users di arsitektur baru |
| **5** | 1 minggu | Cleanup IpcComService SASE-related | Surface area bersih |
| **Total** | **~7 minggu** | | Production fully migrated |

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

3. **Verifikasi schema `user_data` production (read-only — JANGAN ubah):**

   ```sql
   -- Login ke Supabase SQL editor sebagai role yang punya akses read.
   -- HARUS TIDAK ADA migration / ALTER TABLE / CREATE POLICY yang dijalankan.

   -- Konfirmasi kolom yang relevan ada
   SELECT column_name, data_type
   FROM information_schema.columns
   WHERE table_schema='public' AND table_name='user_data'
     AND column_name IN ('uid','configuration','assigned_ip','sase_slice_id','sase_version');

   -- Konfirmasi RLS aktif + policy "Allow all usage"
   SELECT relrowsecurity FROM pg_class WHERE relname='user_data';
   SELECT polname, polcmd FROM pg_policy WHERE polrelid='public.user_data'::regclass;

   -- Inspect format configuration di sample user
   SELECT length(configuration), position('[Interface]' in configuration),
          position('[Peer]' in configuration), position('PrivateKey' in configuration)
   FROM user_data WHERE configuration IS NOT NULL LIMIT 5;
   ```

4. Konfirmasi dengan ops:
   - Format `configuration` adalah INI string standar WireGuard (sudah dikonfirmasi: ya)
   - PrivateKey termasuk di `configuration` (sudah dikonfirmasi: ya)
   - PSK / MTU / Keepalive di production (sudah dikonfirmasi: tidak ada)
   - Realtime tidak aktif (sudah dikonfirmasi: tidak)

### Exit criteria

- [ ] Mapping IPC → tujuan baru lengkap
- [ ] Schema `user_data` di-verify (TIDAK diubah)
- [ ] Format `configuration` confirmed dengan ops
- [ ] User test punya `configuration` ter-populate
- [ ] Sign-off security tentang trust model (PrivateKey ada di Supabase, lihat Bab 7 §7.4)

> **Phase 1 schema migration di-DROP.** Tidak ada migration yang akan dijalankan ke Supabase. Lanjut ke Phase 1 (Helper Service).

## 10.4 Phase 1 — Hermes Helper Service

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

## 10.5 Phase 2 — Client side: `HelperServiceClient` + `SaseConfigClient`

### Tugas

1. Buat folder `HermesNetwork/Sase/Helper/` + `HermesNetwork/Sase/Config/`
2. Implementasi `HelperServiceClient` (named-pipe / unix-socket)
3. Implementasi `SaseConfigClient` (passthrough INI dari `user_data.configuration`; tidak ada KeyStore karena admin yang manage keypair)
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
- [ ] PostgREST query ke `user_data` dengan JWT user mengembalikan row dengan `configuration` valid (sanity check via curl)
- [ ] PR merged

## 10.6 Phase 3 — `SaseConnectionService` + UI

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
       // Tidak ada KeyStore — keypair di-manage admin di user_data.configuration
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

## 10.7 Phase 4 — Beta + 100% rollout

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

Push event ke app log lokal (tabel `sase_audit_log` tidak ada — schema dilarang diubah). Kalau perlu sentralisasi audit, pakai `LogReportService` yang sudah ada (lihat bab 7 §7.10):

- `connect.attempt`, `connect.success`, `connect.timeout`, `connect.error`
- `disconnect.user`, `disconnect.unexpected`
- `reconnect.auto.success`, `reconnect.auto.fail`
- `helper.rpc.error.<code>`

Query Supabase periodic untuk monitor.

### Exit criteria

- [ ] 100% rollout sukses 1 minggu tanpa rollback
- [ ] Connect success rate ≥ 95%
- [ ] No user complaint signifikan

## 10.8 Phase 5 — Cleanup

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

## 10.9 Rollback strategy per phase

| Phase | Rollback |
|---|---|
| 0 | N/A (audit only — tidak ada perubahan) |
| 1 | Uninstall Helper Service via installer; revert PR |
| 2 | Revert PR; arsitektur lama tetap pakai IPC |
| 3 | Toggle feature flag `false` |
| 4 | Hot-fix release flag default `false` |
| 5 | Restore kode lama dari git history |

> Tidak ada rollback DB karena tidak ada DB change.

## 10.10 Risk register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| Helper Service tidak terinstall di endpoint user | Medium | High | Bundle di installer, check Ping di startup |
| Permission issue caller authentication | Medium | Medium | Comprehensive testing multi-user scenarios |
| `configuration` di `user_data` format inkonsisten antar user (mis. ada user yang INI tidak valid) | Medium | High | Validator di `SaseConfigClient.GetFullAsync()` cek `[Interface]` + `[Peer]` ada; fail early dengan pesan jelas. Hubungi admin untuk fix populate. |
| Apple Developer ID expire | Low | Critical | Calendar reminder 30 hari |
| Conflict dengan VPN client lain (NordVPN, dll.) | Medium | Medium | Document incompatibility, detect saat startup |
| Kill switch terlalu agresif → user lock out | Medium | High | Default soft-fail; opt-in kill switch per role |
| Realtime subscription tidak available di self-hosted | Medium | Low | Fallback ke polling 5 menit |

## 10.11 Definition of Done

Migrasi **selesai** kalau:

- [ ] Phase 0–6 semua exit criteria tercentang
- [ ] `LegacySaseConnectionService` dihapus dari main
- [ ] Feature flag `UseNewSaseStack` dihilangkan
- [ ] Telemetry produksi 2 minggu menunjukkan no increase di error rate
- [ ] Onboarding doc tim engineering update
- [ ] Tidak ada call IPC custom yang menyentuh SASE
- [ ] Helper Service signed + notarized untuk Mac, Authenticode signed untuk Win
- [ ] **Konfirmasi: TIDAK ADA migration / `ALTER TABLE` / trigger / policy baru yang dijalankan ke Supabase production.** Schema `user_data` tetap apa adanya.

---

[← Bab 9 CLI Reference]({{ site.baseurl }}{% link docs/09-cli-reference.md %}){: .btn }
[Bab 11 — Troubleshooting →]({{ site.baseurl }}{% link docs/11-troubleshooting.md %}){: .btn .btn-primary }
