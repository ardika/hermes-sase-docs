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

### **Kenapa tetap pakai WireGuard? Bukan switch ke OpenZiti / Tailscale / NetBird?**

Kekuatan WireGuard:
- Kernel-mode (Win/Linux) atau NetworkExtension-grade (Mac) — sangat cepat
- Crypto modern, audit clean (3500 LoC)
- Sudah running di Hermes, terbukti stabil
- Format config sederhana, mudah di-debug

Yang ditambahkan dokumen ini = **control plane + key management** di atas WireGuard. Hasilnya: experience setara Tailscale (managed, identity-aware, auto-rotate) tanpa lock-in vendor.

Migrasi data plane ke teknologi lain = scope berbeda dengan trade-off besar. Sekarang fokus refactor cara kita mengelola WireGuard.

### **Kenapa pakai Edge Function sebagai trust boundary, bukan langsung dari client?**

Sama persis seperti alasan di TRMM:
- API key SASE control plane = privileged, bisa register/revoke peer apa pun
- Embed di binary client = sekali leak, semua peer di-impact
- Edge function = key di server only, scope kerusakan terbatas

### **Kenapa tidak pakai NetworkExtension framework di Mac?**

NetworkExtension butuh **Special Entitlement** dari Apple yang sulit didapat untuk app enterprise kustom. Userspace `wireguard-go` + LaunchDaemon cukup untuk Hermes scale. Migrasi ke NetworkExtension = future work kalau perlu kernel-mode atau MDM-managed VPN.

### **Apa yang terjadi kalau Edge Function down?**

- ❌ User baru tidak bisa connect (butuh request config)
- ❌ Config tidak bisa di-refresh
- ✅ Tunnel yang sudah up tetap jalan sampai TTL config (24 jam)
- ✅ Setelah Edge Function recover, semua lanjut normal

UI design: tampilkan banner warning kalau config expire <2 jam dan refresh gagal.

### **Apa yang terjadi kalau gateway WireGuard down?**

- Tunnel tidak handshake, status "Faulted" di UI
- ConnectionService coba reconnect 3 kali dengan back-off
- Setelah itu, user harus klik Connect manual lagi
- Edge Function masih jalan (independen dari gateway)

Failover gateway: ops update `SASE_GATEWAY_ENDPOINT` di Supabase secrets → user dapat endpoint baru di config refresh berikutnya.

### **Apakah perlu kill switch?**

Tergantung role:
- **Executive / role sensitif**: ya, kill switch on (no traffic kecuali via tunnel)
- **Engineer / general**: opsional, biasanya off (split-tunnel cukup)
- **Contractor**: ya, kill switch on

Default soft-fail; opt-in kill switch per role policy di Edge Function.

## 12.2 Pertanyaan implementasi

### **Boleh pakai library .NET WireGuard wrapper (mis. WireGuard.NET) bukan shell out ke `wg`?**

Bisa, tapi:
- Library wrapper biasanya outdated atau partial
- Shell out ke `wg.exe` / `wg-quick` adalah path resmi WireGuard project
- Output `wg show dump` parse-able dan stable
- Mengurangi NuGet dependency

Hanya pakai library kalau ada kebutuhan spesifik (mis. embed userspace WireGuard ke binary tanpa external CLI).

### **Bisa nggak pakai gRPC untuk Edge Function?**

Supabase Edge Functions hanya support HTTP. Untuk gRPC, deploy ke Cloud Run / Fly.io / dll. Untuk scope ini, REST cukup.

### **Kalau Hermes deploy beberapa gateway (multi-region), bagaimana pilih yang dekat?**

Edge Function bisa pilih gateway terdekat berdasarkan:
- IP geolocation client (request `req.headers.get("x-forwarded-for")`)
- User role / preference
- Health check gateway

Implementasi: tambah field `gateway_pool` di env, pilih satu, return endpoint-nya.

```typescript
const gateways = JSON.parse(Deno.env.get("SASE_GATEWAY_POOL") || "[]");
const closest = pickByGeo(req, gateways);
```

### **Bisa nggak active-passive multi-tunnel (failover) di client?**

WireGuard mendukung multiple peer di satu interface, tapi tidak punya built-in failover logic. Untuk active-passive:
- Tetap 1 tunnel, 1 gateway peer
- Edge Function pilih gateway saat config refresh
- Kalau primary down, ops update endpoint → client refresh → switch

Lebih sederhana dan reliable daripada client-side failover.

### **Performance overhead per tunnel?**

WireGuard overhead minimal:
- Per packet: ~32 byte header tambahan
- CPU: 1–5% per Gbps di laptop modern (AES-NI / hardware crypto)
- Latency: +1–3 ms (kalau gateway dekat)

Untuk Hermes use case (ribuan endpoint), bottleneck ada di gateway server, bukan client.

### **Apakah perlu real-time WebSocket untuk push policy update?**

Phase 1: tidak. Polling 5 menit cukup.

Phase 2 (kalau perlu): tambah Supabase Realtime subscription. Kalau row di `user_profiles` user X update, push notification ke desktop client → trigger refresh.

```typescript
const channel = supabase.channel('user-' + userId)
  .on('postgres_changes', {
    event: 'UPDATE', schema: 'public', table: 'user_profiles',
    filter: `id=eq.${userId}`
  }, payload => {
    if (payload.new.sase_role !== payload.old.sase_role) {
      triggerRefresh();
    }
  })
  .subscribe();
```

## 12.3 Pertanyaan operasional

### **Berapa cost untuk SASE setup ini?**

Cost utama:
- Gateway VPS (4–8 vCPU, sudah ada): $40–80/bulan
- Bandwidth: tergantung volume, mostly included
- SASE control plane (kalau pakai vendor): $X/peer/bulan
- Supabase Edge Function: included di plan free / pro
- Apple Developer ID: $99/tahun

Untuk 1000 endpoint dengan WireGuard self-hosted: ~$50–100/bulan.

### **Berapa peer maximum per gateway?**

WireGuard self-hosted di VPS 4 vCPU 8 GB RAM bisa handle ~10,000 peer dengan handshake load normal. Bottleneck biasanya bukan WireGuard, tapi:
- NAT table di network (gateway behind NAT)
- DNS resolver
- iptables rules

Untuk 50,000+ peer, scale horizontal: multiple gateway, peer-aware routing.

### **Berapa lama config refresh berlaku?**

Default 24 jam (set di Edge Function). Bisa di-tune:
- Pendek (1 jam): rotate PSK lebih sering, lebih aman, lebih banyak request
- Panjang (7 hari): lebih sedikit request, kurang sering rotate

Sweet spot: 24 jam = balance antara security dan ops cost.

### **Bagaimana revoke akses cepat (compromised user)?**

3 langkah:

```sql
-- 1. Mark peer revoked di DB (Edge Function refuse return config)
UPDATE sase_peer SET status = 'revoked' WHERE user_id = '<uuid>';
```

```bash
# 2. Hapus peer di gateway (drop traffic immediately)
curl -X DELETE "$SASE_API_URL/peers/<pubkey>" \
  -H "Authorization: Bearer $SASE_API_KEY"
```

```sql
-- 3. Suspend user di Supabase Auth (cegah re-enroll)
UPDATE auth.users SET banned_until = '2099-12-31' WHERE id = '<uuid>';
```

Tunnel user disconnect dalam ~30 detik.

### **Bagaimana cara onboard customer baru (multi-tenant)?**

Untuk multi-tenant, tambah `tenant_id` di tabel `sase_peer` dan policy mapping:

```sql
ALTER TABLE user_profiles ADD COLUMN tenant_id UUID;

ALTER TABLE sase_peer ADD COLUMN tenant_id UUID;
```

Edge Function panggil control plane dengan tag tenant:

```typescript
await registerPeerAtGateway({
  ...
  tags: [role, `tenant:${tenantId}`],
});
```

Gateway side: configure routing/firewall per tenant tag.

## 12.4 Pertanyaan keamanan

### **WireGuard sudah aman by default. Kenapa repot tambah PSK + key rotation?**

Defense in depth.

WireGuard crypto memang strong. Tapi:
- PSK = mitigasi kalau private key bocor (memory dump, malware)
- PSK = future-proof terhadap quantum attack pada Curve25519
- Auto rotation = limit window of damage kalau ada compromise

Cost tambah PSK + rotation: ~50 baris kode di Edge Function. Benefit: significant.

### **Apakah private key WireGuard bisa di-extract dari Windows DPAPI?**

Hanya kalau attacker:
1. Sudah dapat akses sebagai user yang sama (laptop unlocked atau credential bocor)
2. Atau punya akses fisik + DPAPI master key

Bukan defense terhadap state actor, tapi cukup untuk mayoritas threat (laptop curi, malware umum).

Untuk paranoid mode, store private key di **TPM / Secure Enclave** — future work, butuh refactor KeyStore.

### **Apakah PSK menggantikan kebutuhan HTTPS?**

Tidak. PSK protect data plane (tunnel UDP). HTTPS protect control plane (request config). Dua-duanya complementary.

### **Apakah ada audit log yang bocor PII?**

Tabel `sase_audit_log` log:
- user_id (UUID, bukan email)
- device_id (hash)
- role
- hostname (potentially identifying)
- timestamp

Untuk GDPR compliance, hostname bisa di-hash juga atau di-redact setelah retention period.

### **Apakah TRMM agent bisa lihat trafik SASE?**

Tidak — TRMM agent jalan di endpoint user, melihat **interface lokal**. Trafik di interface tunnel sudah encrypted di luar process app.

Tapi: TRMM agent **bisa** lihat plaintext trafik di interface non-tunnel kalau pakai script monitoring. Out of scope dokumen ini, tapi tetap relevan untuk threat model.

### **Bagaimana dengan dual-VPN (user pakai personal VPN + Hermes SASE)?**

WireGuard biasanya jalan baik berlapis dengan VPN lain (mis. corporate over personal NordVPN). Yang harus diwaspadai:
- MTU stacking → set tunnel Hermes MTU lebih kecil (1280)
- Routing conflict → Hermes AllowedIPs jangan overlap dengan personal VPN

## 12.5 Roadmap & future work

| Fitur | Estimasi effort | Value |
|---|---|---|
| WebSocket push policy update (Realtime) | 1 sprint | Medium — hilangkan polling lag |
| Per-app VPN (split tunnel by process) | 4+ sprint | High — UX better, complex implementasi |
| TPM / Secure Enclave key storage | 2 sprint | Medium — security improvement |
| Multi-region gateway selection | 1 sprint | Medium — performance untuk global users |
| Mobile (iOS/Android) | 4+ sprint | High — but di luar scope desktop |
| Built-in network throughput analytics | 2 sprint | Low — nice-to-have |

## 12.6 Pertanyaan komparasi

### **Beda arsitektur ini sama Tailscale apa?**

| Aspek | Tailscale | Refactor SASE Hermes |
|---|---|---|
| Data plane | WireGuard (modified) | WireGuard (vanilla) |
| Control plane | Tailscale cloud | Self-hosted Edge Function + control plane |
| Identity | OAuth (Google, GitHub, dll.) | Supabase Auth |
| Mesh / hub-spoke | Mesh peer-to-peer | Hub-spoke (semua via gateway) |
| Open source | Server: only Headscale (community) | Full open source |
| Cost | $5–20/user/bulan | Self-hosted, ~$0.05/user/bulan |

Hermes refactor = essentially "Tailscale-style management dengan vanilla WireGuard data plane, full self-hosted".

### **Beda dengan NetBird?**

NetBird sudah punya semua yang kita build di atas (managed WireGuard, identity-aware, self-hosted). Kalau team ops Hermes mau lighten scope: switch ke NetBird sebagai control plane saja.

Trade-off:
- NetBird = built-in, tested, less code to maintain
- Custom = full control, integrasi lebih dalam dengan Supabase + TRMM stack

Saat ini Hermes pilih custom karena sudah investasi di Supabase ecosystem.

---

## Pertanyaan tidak ada di sini?

Buka issue di repo `Hermes-Network-Inc/HermesNetwork360Guard` atau Slack `#engineering`.

---

[← Bab 11 Troubleshooting]({{ site.baseurl }}{% link docs/11-troubleshooting.md %}){: .btn }
[← Kembali ke Beranda]({{ site.baseurl }}{% link index.md %}){: .btn .btn-primary }
