---
layout: default
title: 12. FAQ
nav_order: 13
permalink: /docs/faq/
---

# 12. FAQ
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 12.1 Pertanyaan arsitektur

### **Kenapa harus ada Hermes Helper Service terpisah? Tidak bisa UI langsung manage WireGuard?**

Dua alasan:

1. **`HermesNetwork360Guard.exe` jalan as user (bukan admin)** — operasi WireGuard butuh admin/root (install Win Service, write ke `C:\Program Files\WireGuard\Data\`, `sc start/stop`).
2. **UAC dialog tiap operasi = UX rusak.** Helper sekali install (oleh installer dengan elevation), terus jalan sebagai service persistent.

Ini essentially menggantikan ServiceEngine.exe lama, tapi dengan **scope minimal** (hanya verb privileged terkait WireGuard) dan **kontrak typed JSON-RPC** (bukan magic strings).

### **Kenapa tidak pakai Edge Function seperti di TRMM docs?**

**Supabase yang dipakai self-hosted, dan self-hosted Supabase tidak include Edge Functions.** Self-hosted hanya support:
- GoTrue auth
- PostgREST (direct REST ke Postgres)
- Realtime
- Storage
- (Tidak ada Deno runtime untuk Edge Functions)

Solusi: pakai PostgREST + RLS langsung dari client. Untuk logic server-side yang lebih complex, deploy REST service standalone (FastAPI / Express / dll) terpisah.

### **Kenapa konfigurasi WireGuard disimpan di Supabase `user_data.configuration`, bukan di gateway control plane?**

Karena **konfigurasi WireGuard per-user sudah ada di kolom `user_data.configuration` (text, full INI string)** sebagai source of truth (existing setup di production Hermes). Admin/automation populate config per-user, client baca dan apply via Helper.

Tidak ada "central gateway" tunggal yang di-hardcode di kode — tiap user bisa punya endpoint berbeda, AllowedIPs berbeda, dll. Semua di-define per-user di Supabase oleh admin tooling (di luar scope dokumen ini).

### **Bagaimana kalau client mau pakai per-device WG keypair (bukan dari Supabase)?**

Pattern itu **lebih aman secara threat model**, tapi **TIDAK didukung di scope dokumen ini** karena:

1. **Schema Supabase tidak boleh diubah** — kolom `wg_public_key` tidak ada di production, tidak akan ditambahkan.
2. **Admin tooling existing menempatkan PrivateKey langsung di `configuration`** — pattern "client publish pubkey, admin re-issue" akan butuh perubahan tooling admin di luar scope.

Implementasi client mengikuti realita production: PrivateKey datang dari Supabase, client passthrough.

Kalau ke depan Hermes mau migrasi ke pattern keypair-per-device, butuh:
- Schema change di Supabase (tambah kolom pubkey writeback)
- Update admin tooling supaya generate config tanpa PrivateKey + register pubkey saat publish
- Update client supaya generate keypair lokal + inject

Itu adalah project terpisah, bukan bagian dokumen ini.

### **Kenapa tetap pakai WireGuard? Bukan switch ke Tailscale / NetBird / OpenZiti?**

WireGuard kekuatan:
- Kernel-mode (Win/Linux), NetworkExtension-grade (Mac)
- Crypto modern, audited
- Sudah running di Hermes
- Format config sederhana

Yang ditambahkan refactor ini = **control plane di sisi Supabase** + **Helper Service untuk lifecycle**. Hasilnya: managed WireGuard tanpa lock-in vendor.

Migrasi ke teknologi lain = scope berbeda, trade-off besar. Sekarang fokus refactor cara kita mengelola WireGuard.

### **Apa yang terjadi kalau Supabase down?**

- ❌ User baru tidak bisa connect (butuh read user_data)
- ❌ Config tidak bisa di-refresh
- ✅ Tunnel yang sudah up tetap jalan (Helper independent)
- ✅ UI tetap menampilkan status terakhir dari Helper

UI design: tampilkan banner warning kalau read user_data gagal terus-menerus.

### **Apa yang terjadi kalau Helper crash?**

- ❌ UI tidak bisa request ApplyConfig / Start / Stop
- ✅ Tunnel yang sudah up tetap jalan (Helper hanya manage, tidak in-line dengan data path)
- UI tampilkan banner "Helper Service not responding"
- Auto-restart oleh systemd (Mac LaunchDaemon `KeepAlive=true`) atau Windows Service Recovery

### **Bagaimana kalau user pindah jaringan (WiFi → 4G)?**

1. WireGuard self-heal via `PersistentKeepalive` (25 detik)
2. `NetworkChange.NetworkAddressChanged` event di UI trigger pemeriksaan handshake
3. Kalau handshake stale > 2 menit, force `Disconnect → Connect`

Detail di [Bab 6]({{ site.baseurl }}{% link docs/06-connection-flow.md %}) §6.6.

### **Apakah perlu kill switch?**

Tergantung role:
- **Executive / role sensitif**: ya, kill switch on
- **Engineer / general**: opsional, biasanya off
- **Contractor**: ya, kill switch on

Default soft-fail; opt-in kill switch per role. Implementasi via firewall rule yang Helper apply saat tunnel up. Lihat [Bab 7]({{ site.baseurl }}{% link docs/07-keamanan.md %}) §7.7.

## 12.2 Pertanyaan implementasi

### **Boleh pakai library .NET WireGuard wrapper bukan shell out?**

Bisa, tapi:
- Library wrapper biasanya outdated atau partial coverage
- Shell out ke `wireguard.exe`/`wg`/`wg-quick` adalah path resmi
- Output `wg show dump` parse-able dan stable
- Mengurangi NuGet dependency

Hanya pakai library kalau ada kebutuhan spesifik (mis. embed userspace WireGuard tanpa external CLI).

### **Boleh pakai gRPC untuk IPC UI ↔ Helper?**

Bisa, tapi overkill. Named-pipe + JSON-RPC sudah cukup karena:
- Kontrak verb sederhana (~6-8 method)
- Built-in OS authentication via pipe / socket creds
- Ringan, tidak butuh code generation tooling
- Standard pattern di Windows untuk service ↔ user app

gRPC bagus untuk RPC lintas mesin / lintas bahasa. Untuk lokal IPC, named-pipe lebih simple.

### **Bagaimana kalau perlu verb baru di Helper (mis. enable kill switch)?**

Tambah ke whitelist:
1. Definisikan params record di `Program.cs` Helper
2. Tambah handler di `JsonRpcServer.DispatchAsync`
3. Tambah implementation di `IOsBackend` + Win/Mac
4. Tambah method di `IHelperServiceClient` di UI app
5. Bump JSON-RPC version untuk audit trail

### **Apakah perlu real-time WebSocket untuk update `configuration`?**

Tidak — di production Hermes, **Realtime tidak aktif** untuk semua tabel (verified via Supabase dashboard, semua "Realtime Enabled = ❌"). Polling tiap 5 menit cukup untuk UX baik (admin update jarang).

Kalau ke depan Realtime di-enable di Supabase, client bisa di-extend untuk subscribe ke row `user_data` user. Tapi itu opsional, bukan perubahan yang diperlukan saat ini.

### **Bagaimana kalau perlu support Linux?**

`IOsBackend` adalah interface — tinggal tambah `LinuxBackend` yang pakai `systemctl` + `wg-quick`:

```csharp
public sealed class LinuxBackend : IOsBackend
{
    public Task StartTunnelAsync(string name, CancellationToken ct = default)
        => RunAsync("systemctl", $"start wg-quick@{name}", ct);
    // ...
}
```

Avalonia sudah support Linux. WireGuard juga jalan di Linux (kernel module). Tidak ada blocker, hanya scope work tambahan.

### **Bagaimana cara test kalau saya tidak punya Helper di-install?**

Run Helper sebagai console app di terminal admin/sudo:

```powershell
# Windows (admin terminal)
.\publish\HermesHelperSvc.exe
```

```bash
# Mac (sudo)
sudo /usr/local/bin/HermesHelperSvc
```

Di mode console, Helper tetap listen di pipe/socket. UI bisa connect normal.

## 12.3 Pertanyaan operasional

### **Berapa cost SASE setup ini?**

- Gateway WireGuard VPS: $40–80/bulan (existing)
- Supabase self-hosted: cost server existing + storage
- Apple Developer ID: $99/tahun (untuk signing Mac Helper)
- Authenticode cert (Win signing): $200–400/tahun

Untuk 1000 endpoint, cost incremental ~$30–50/bulan (signing cost amortized).

### **Berapa peer maximum per gateway?**

WireGuard self-hosted di VPS 4 vCPU 8 GB RAM bisa handle ~10,000 peer dengan handshake load normal. Bottleneck biasanya:
- NAT table di network (gateway behind NAT)
- DNS resolver
- iptables rules

Untuk 50,000+ peer, scale horizontal: multiple gateway.

### **Berapa lama config refresh berlaku?**

Default 24 jam (set saat config di-generate oleh admin tooling). Bisa di-tune:
- Pendek (1 jam): admin tooling refresh `configuration` lebih sering, mungkin re-issue keypair tiap jam — beban admin tooling tinggi
- Panjang (7 hari): lebih sedikit work, kurang sering rotate

Sweet spot: 24 jam.

### **Bagaimana revoke akses cepat (compromised user)?**

3 langkah:

```sql
-- 1. (Admin tooling) Clear configuration agar client refuse to connect
--    NOTE: hanya admin/service_role yang melakukan ini, bukan client.
UPDATE user_data SET configuration = NULL WHERE uid = '<auth-user-uuid>';
```

```bash
# 2. Hapus peer di gateway (drop traffic immediately)
ssh gateway "wg set wg0 peer <pubkey> remove"
```

```sql
-- 3. Suspend user di Supabase Auth
UPDATE auth.users SET banned_until = '2099-12-31' WHERE id = '<uuid>';
```

Tunnel user disconnect dalam ~30 detik (Helper detect handshake stale → faulted → user notified).

### **Bagaimana onboard customer baru (multi-tenant)?**

Untuk multi-tenant, tambah `tenant_id` di `user_data`:

```sql
ALTER TABLE user_data ADD COLUMN tenant_id UUID;
```

Admin tooling provision per-user `configuration` dengan gateway/AllowedIPs sesuai tenant. Client tidak perlu tahu — dia hanya read `user_data.configuration` user-nya sendiri (RLS enforce).

## 12.4 Pertanyaan keamanan

### **Apakah pakai PSK?**

Tidak — sample 5 user real di production menunjukkan **`PresharedKey` tidak ada** di `configuration`. Production saat ini tidak pakai PSK.

PSK akan menambah defense-in-depth (mitigasi quantum, perlindungan kalau private key bocor), tapi cost tambah PSK = admin tooling harus generate + populate ke `configuration` semua user. Itu keputusan ops, bukan client.

Kalau ke depan admin tooling tambah `PresharedKey = ...` di `[Peer]` section, **client otomatis support tanpa code change** — passthrough INI ke Helper, WireGuard handle PSK natively.

### **Apakah private key WG bisa di-extract dari Windows DPAPI?**

Hanya kalau attacker:
1. Sudah dapat akses sebagai user yang sama (laptop unlocked / credential bocor)
2. Atau punya akses fisik + DPAPI master key

Bukan defense terhadap state actor, tapi cukup untuk mayoritas threat.

Untuk paranoid mode: store private key di TPM / Secure Enclave — future work, butuh perubahan admin tooling (generate keypair via TPM) plus client-side hardening. Tidak applicable sekarang karena PrivateKey datang dari Supabase.

### **Apakah ada audit log yang bocor PII?**

App log lokal (Hermes Guard `app.log`) + OS log (Event Viewer / `os_log`) berisi:
- `user_id` (UUID)
- `action`
- `detail` JSONB (bisa include hostname, public_ip)
- `created_at`

Untuk GDPR compliance, hostname bisa di-hash atau di-redact setelah retention period.

### **Apakah private key WG di Supabase aman?**

**Production saat ini: `configuration` di Supabase include PrivateKey.** Itu berarti trust model "admin yang sama yang juga manage user". Acceptable kalau admin = orang yang sama, tapi sub-optimal dari threat model perspective.

**Pattern yang lebih aman (BUKAN scope dokumen ini, butuh schema change Supabase)**:
1. Client generate keypair sendiri
2. Publish hanya pubkey ke kolom baru (perlu schema change)
3. Admin tooling generate config tanpa PrivateKey
4. Client inject PrivateKey lokal saat apply

Detail di [Bab 5]({{ site.baseurl }}{% link docs/05-config-service.md %}) §5.2.1 dan [Bab 7]({{ site.baseurl }}{% link docs/07-keamanan.md %}) §7.4.

### **Bisakah user lihat `configuration` user lain?**

Tidak. RLS sudah aktif di `user_data` production (verified):

```sql
-- Sudah ada di production (jangan dibuat ulang).
-- Policy "Allow all usage" dengan rule:
--   USING (uid() = uid)
--   WITH CHECK (uid() = uid)
-- 
-- uid() = Supabase auth helper yang return auth.users.id dari JWT.
-- Kolom user_data.uid = uuid foreign key ke auth user.
```

Client query dengan JWT user → PostgREST + RLS filter ke row dengan `uid` matching user otomatis.

### **Apakah Helper bisa di-eksploitasi untuk arbitrary command execution?**

Tidak, **kalau implementasi follow whitelist**:
- Whitelist verb saja (Ping, ApplyConfig, Start, Stop, Status, Install, Uninstall)
- Validate input regex (tunnel name)
- Validate config sintaks (must have `[Interface]` + `[Peer]`)
- No shell-out generic command

Kalau implementator tambah verb "RunCommand" atau pass user input langsung ke shell — itu vulnerability. Code review wajib untuk tiap verb baru.

### **Bagaimana dengan dual-VPN (user pakai personal VPN + Hermes SASE)?**

Bisa berlapis tapi waspadai:
- MTU stacking → set Hermes MTU lebih kecil (1280)
- Routing conflict → AllowedIPs jangan overlap dengan personal VPN
- Performance degradation karena double encryption

## 12.5 Roadmap & future work

| Fitur | Estimasi | Value |
|---|---|---|
| Realtime push update via Supabase Realtime | 1 sprint | Medium |
| Per-app VPN (split by process) | 4+ sprint | High |
| TPM / Secure Enclave key storage | 2 sprint | Medium |
| Multi-region gateway selection | 1 sprint | Medium |
| Mobile (iOS/Android) | 4+ sprint | High (out of scope desktop) |
| Built-in throughput analytics dashboard | 2 sprint | Low |
| Ditch ServiceEngine.exe sepenuhnya (setelah TRMM + SASE migrate) | 1 sprint | High (security/maintenance) |

## 12.6 Komparasi dengan TRMM doc

| Aspek | TRMM doc | SASE doc (ini) |
|---|---|---|
| Backend | Supabase Edge Function (managed Supabase asumsi) | PostgREST direct (self-hosted) |
| Privileged op | Tidak butuh helper (TRMM API calls) | Helper Service required |
| Custom IPC | Dihapus | Diganti typed JSON-RPC ke Helper |
| Trust boundary | Edge Function pegang TRMM API key | RLS Postgres + Helper local auth |
| Scope helper | N/A | Verb whitelist khusus WireGuard |

Kedua dokumen pakai pola **3-lapis** yang konsisten supaya tim engineering punya mental model sama untuk seluruh stack refactor.

---

## Pertanyaan tidak ada di sini?

Buka issue di repo `Hermes-Network-Inc/HermesNetwork360Guard` atau Slack `#engineering`.

---

[← Bab 11 Troubleshooting]({{ site.baseurl }}{% link docs/11-troubleshooting.md %}){: .btn }
[← Kembali ke Beranda]({{ site.baseurl }}{% link index.md %}){: .btn .btn-primary }
