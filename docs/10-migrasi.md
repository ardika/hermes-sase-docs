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

Sama seperti refactor TRMM, migrasi SASE dilakukan **paralel** — arsitektur baru jalan berdampingan dengan kode lama. Setelah arsitektur baru terbukti stabil di produksi, kode lama dihapus.

Manfaat:

- **Rollback gampang** — toggle feature flag
- **Tidak ada downtime user** — VPN tetap kerja selama migrasi
- **Bisa bertahap** — refactor satu OS dulu, baru lanjut OS lain
- **Tim paralel** — backend (Edge Function) tidak block frontend

## 10.2 Phase overview

| Phase | Durasi target | Aktivitas | Hasil terlihat |
|---|---|---|---|
| **0** | 1 minggu | Audit code SASE existing | Dokumen state machine + dependency graph |
| **1** | 1 minggu | Setup Edge Function + DB schema | Edge function deploy, ter-test isolated |
| **2** | 2 minggu | Implementasi `TunnelSupervisor` (Win + Mac) | Standalone test, belum integrate UI |
| **3** | 1 minggu | Implementasi `SaseConfigClient` + KeyStore | Standalone, belum integrate |
| **4** | 1 minggu | `ConnectionService` + UI binding | Tab SASE baru pakai arsitektur baru, masih opt-in via flag |
| **5** | 1 minggu | Beta + 100% rollout | Production users di arsitektur baru |
| **6** | 1 minggu | Cleanup IpcComService SASE-related code | Surface area bersih |
| **Total** | **~8 minggu** | | Production fully migrated |

## 10.3 Phase 0 — Audit

### Tujuan

Inventarisasi state SASE saat ini.

### Tugas

1. Search occurrence di repo:

   ```powershell
   cd F:\Upwork\CharlesUSA\HermesNetwork360-Avalonia
   git grep -nE "WireGuard|wireguard|SASE|sase" -- '*.cs' > audit-sase-callsites.txt
   git grep -n '"Code".*"SASE"' -- '*.cs' >> audit-sase-callsites.txt
   ```

2. Catat di tabel:

   | File:Line | Code IPC | Operasi | Tujuan baru |
   |---|---|---|---|
   | `ConfigViewModel.cs:1555` | `Z1398V` SASE Start | Start tunnel | `ConnectionService.ConnectAsync()` |
   | `ConfigViewModel.cs:390` | (sc query) | Get state | `ConnectionService.RefreshStatusAsync()` |
   | `ConfigViewModel.cs:716` | `XDR Start` | XDR (di luar scope SASE) | (lihat doc lain) |

3. Dokumentasikan **konfigurasi WireGuard sekarang**:
   - Format `.conf` di-install di mana?
   - Public key gateway saat ini
   - AllowedIPs saat ini
   - Apakah pakai PSK?
   - Apakah pakai DNS push?

4. Mapping role → policy (lihat Bab 3 §3.1.2)

### Deliverable

`audit-sase.md` di repo internal, berisi:
- Daftar file kode existing yang menyentuh SASE
- Konfigurasi WireGuard saat ini
- Mapping role → policy yang disetujui ops

### Exit criteria

- [ ] Semua call site SASE teridentifikasi
- [ ] Konfigurasi sekarang ter-dokumentasi
- [ ] Sign-off ops & security tentang policy mapping

## 10.4 Phase 1 — Edge Function + DB schema

### Tujuan

`sase-config` Edge Function siap di staging Supabase, ter-test, belum dipakai client.

### Tugas

1. Apply DB migration:

   ```bash
   supabase migration new add_sase_tables
   # paste SQL dari Bab 5 §5.2
   supabase db push
   ```

2. Tambah env secret:

   ```bash
   supabase secrets set SASE_API_URL=...
   supabase secrets set SASE_API_KEY=...
   supabase secrets set SASE_GATEWAY_ENDPOINT=n1.ndr24.com:51820
   supabase secrets set SASE_GATEWAY_PUBLIC_KEY=...
   supabase secrets set SASE_DEFAULT_DNS=10.0.0.1,1.1.1.1
   ```

3. Deploy edge function (lihat Bab 5 §5.7):

   ```bash
   supabase functions new sase-config
   # tulis index.ts
   supabase functions deploy sase-config
   ```

4. Test manual via curl:

   ```bash
   JWT=$(...)  # dapatkan JWT user test
   curl -X POST $SB_URL/functions/v1/sase-config \
     -H "Authorization: Bearer $JWT" \
     -d '{"device_id":"test","public_key":"<wg-pub>","platform":"windows"}'
   # Expected: 200 dengan SaseConfigDto
   ```

5. Test register peer benar-benar muncul di SASE control plane dashboard

### Exit criteria

- [ ] DB migration applied di staging
- [ ] Edge function deployed
- [ ] Manual curl test sukses
- [ ] Peer test muncul di SASE control plane
- [ ] `sase_audit_log` ter-populate

## 10.5 Phase 2 — TunnelSupervisor

### Tujuan

`ITunnelSupervisor` (Win + Mac) terimplementasi, ter-test standalone.

### Tugas

1. Buat folder `HermesNetwork/Sase/Supervisor/` per spec di Bab 4
2. Implementasi `WindowsTunnelSupervisor` + `MacTunnelSupervisor`
3. Unit test dengan `FakeTunnelSupervisor` mock
4. Integration test:

   **Windows:**
   ```powershell
   # Generate config dummy
   $priv = & "C:\Program Files\WireGuard\wg.exe" genkey
   # ... build dummy .conf, manual register di gateway ...

   # Test via supervisor
   dotnet run --project HermesNetwork.SupervisorTest -- apply $config
   dotnet run --project HermesNetwork.SupervisorTest -- start
   & "C:\Program Files\WireGuard\wg.exe" show
   dotnet run --project HermesNetwork.SupervisorTest -- stop
   dotnet run --project HermesNetwork.SupervisorTest -- uninstall
   ```

5. Code review

### Exit criteria

- [ ] CI hijau
- [ ] Unit test coverage >= 80%
- [ ] Integration test sukses Win + Mac
- [ ] PR `feature/sase-tunnel-supervisor` merged

## 10.6 Phase 3 — SaseConfigClient + KeyStore

### Tujuan

Layer 2 lengkap; client bisa request config dari Edge Function dan cache keypair.

### Tugas

1. Implementasi `KeyStore` (Win DPAPI + Mac mode 0600)
2. Implementasi `KeyPairGenerator` (panggil `wg genkey` / `wg pubkey`)
3. Implementasi `DeviceIdHelper`
4. Implementasi `SaseConfigClient`
5. Integration test:

   ```csharp
   [Fact]
   public async Task RequestConfigAsync_ReturnsValidConfig()
   {
       // Setup: real Supabase staging + JWT user test
       var config = await client.RequestConfigAsync();
       config.Peers.Should().HaveCount(1);
       config.Peers[0].PresharedKey.Should().NotBeNullOrEmpty();
   }
   ```

### Exit criteria

- [ ] Test integration end-to-end ke staging Supabase berhasil
- [ ] KeyStore round-trip (save → load) works di Win + Mac
- [ ] PR merged

## 10.7 Phase 4 — ConnectionService + UI

### Tujuan

UI tab SASE pakai arsitektur baru (di balik feature flag).

### Tugas

1. Implementasi `SaseConnectionService` per Bab 6
2. Refactor `SaseTabViewModel` untuk pakai service baru
3. Tambah feature flag di `appsettings.json`:

   ```json
   {
     "FeatureFlags": {
       "UseNewSaseStack": false
     }
   }
   ```

4. Wire DI:

   ```csharp
   if (config.FeatureFlags.UseNewSaseStack)
   {
       services.AddSingleton<ISaseConnectionService, SaseConnectionService>();
       services.AddSingleton<ITunnelSupervisor>(_ => TunnelSupervisorFactory.Create("Hermes"));
       services.AddSingleton<ISaseConfigClient, SaseConfigClient>();
   }
   else
   {
       services.AddSingleton<ISaseConnectionService, LegacySaseConnectionService>();
   }
   ```

5. Test UI di Win + Mac dengan flag `true`

### Exit criteria

- [ ] Toggle flag → swap behavior, no crash
- [ ] Connect → Disconnect → Reconnect cycle bekerja di kedua OS
- [ ] State indicator UI update real-time
- [ ] Background monitor active saat tab dibuka

## 10.8 Phase 5 — Beta + 100% rollout

### Strategi

1. **Internal QA** (1 minggu) — flag `true` di staging, semua tester pakai SASE baru
2. **Beta release** (1 minggu) — 5% production users dapat flag `true`
3. **Monitor**:
   - Connect success rate >= 95%?
   - Avg time-to-connect < 10 detik?
   - Disconnect rate per hour < 1%?
4. **100% rollout** kalau metric OK
5. **Rollback** kalau ada issue:

   ```json
   { "FeatureFlags": { "UseNewSaseStack": false } }
   ```

   Push update kecil yang flip flag → user reload app → kembali ke legacy.

### Telemetry yang di-track

- `sase.connect.attempt` (count)
- `sase.connect.success` (count)
- `sase.connect.duration_seconds` (histogram)
- `sase.connect.failure.{reason}` (count)
- `sase.disconnect.unexpected` (count)
- `sase.handshake.stale_detected` (count)
- `sase.config.refresh.success` (count)
- `sase.config.refresh.failure` (count)

Push ke Supabase `telemetry_events` table atau Sentry.

### Exit criteria

- [ ] 100% rollout sukses 1 minggu tanpa rollback
- [ ] Metric stable
- [ ] Tidak ada user complaint signifikan

## 10.9 Phase 6 — Cleanup

### Tujuan

Hapus kode SASE lama yang sudah tidak terpakai.

### Tugas

1. Verifikasi tidak ada reference legacy:

   ```bash
   git grep "LegacySaseConnectionService"   # harus 0 hits
   git grep '"Code".*"SASE"'                # harus 0 hits
   git grep "ExecuteSaseButton" -- '*.cs'   # 0 hits
   ```

2. Hapus file:

   ```
   HermesNetwork/Service/SaseLegacyHandler.cs
   HermesNetwork/Sase.Old/                  (kalau ada folder lama)
   ```

3. Hapus feature flag `UseNewSaseStack` (selalu true sekarang)
4. Hapus binary path lama yang tidak relevan dari installer
5. Update README + dokumentasi internal

### Exit criteria

- [ ] CI hijau setelah cleanup
- [ ] Release notes mention "SASE rewrite — better key rotation, identity-aware policy"
- [ ] 1 minggu produksi tanpa regression

## 10.10 Rollback strategy per phase

| Phase | Rollback |
|---|---|
| 0 | N/A (audit only) |
| 1 | Drop schema kolom (kalau memungkinkan), undeploy edge function |
| 2 | Revert PR; arsitektur lama tetap pakai IPC |
| 3 | Revert PR |
| 4 | Toggle feature flag false; force release update kalau crash |
| 5 | Hot-fix release dengan flag default false |
| 6 | Restore dari git history (legacy code masih ada di history) |

## 10.11 Risk register

| Risk | Probability | Impact | Mitigation |
|---|---|---|---|
| WireGuard binary belum terinstall di endpoint user | Medium | High | Bundle di installer, check di first run |
| User block UAC dialog (Windows) atau osascript dialog (Mac) | High | Medium | Educate user, retry mechanism |
| SASE control plane API tidak match expected response | High | High | Comprehensive integration test sebelum 100% rollout |
| Config refresh terlalu sering → spike load di Edge Function | Low | Medium | Caching di client, exponential back-off |
| Mac code signing expire | Low | Critical | Calendar reminder 30 hari sebelum |
| Conflict dengan VPN client lain (Cisco AnyConnect, etc.) | Medium | High | Document incompatibility, detection check di startup |
| Kill switch terlalu agresif → user lock out | Medium | High | Default soft-fail; opt-in kill switch per role |

## 10.12 Definition of Done

Migrasi dianggap **selesai** kalau:

- [ ] Phase 0–6 semua exit criteria centang
- [ ] `LegacySaseConnectionService` (atau setara) sudah dihapus dari main branch
- [ ] Feature flag `UseNewSaseStack` sudah dihilangkan
- [ ] Telemetry produksi 2 minggu menunjukkan no increase di error rate
- [ ] Onboarding doc tim engineering update
- [ ] Tidak ada call IPC custom yang menyentuh SASE

---

[← Bab 9 CLI Reference]({{ site.baseurl }}{% link docs/09-cli-reference.md %}){: .btn }
[Bab 11 — Troubleshooting →]({{ site.baseurl }}{% link docs/11-troubleshooting.md %}){: .btn .btn-primary }
