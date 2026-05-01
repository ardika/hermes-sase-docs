---
layout: default
title: Beranda
nav_order: 1
description: "Panduan lengkap arsitektur dan implementasi SASE berbasis WireGuard untuk Hermes Network 360 Guard desktop client."
permalink: /
---

# Panduan Refactor SASE (WireGuard)

**Hermes Network 360 Guard — Desktop Client (Windows & macOS)**
{: .fs-6 .fw-300 }

[Mulai dari Pendahuluan]({{ site.baseurl }}{% link docs/01-pendahuluan.md %}){: .btn .btn-primary .fs-5 .mb-4 .mb-md-0 .mr-2 }
[Lihat Arsitektur]({{ site.baseurl }}{% link docs/02-arsitektur.md %}){: .btn .fs-5 .mb-4 .mb-md-0 }

---

## Tentang Dokumen Ini

Dokumen ini adalah panduan lengkap untuk **merefactor lapisan SASE** (Secure Access Service Edge) di aplikasi desktop **Hermes Network 360 Guard** (`HermesNetwork360-Avalonia`).

Saat ini SASE dijalankan dengan **WireGuard** sebagai data plane, di-supervisi lewat custom IPC ke `ServiceEngine.exe`, dan menggunakan konfigurasi statis yang dihardcode saat install. Pendekatan ini berfungsi tapi punya banyak titik gesek operasional (key rotation manual, tidak ada identity-aware policy, sulit roaming, susah audit).

Dokumen ini menjelaskan **arsitektur tiga-lapis** untuk:

1. **Tetap pakai WireGuard sebagai data plane** (cepat, hemat resource, terbukti)
2. **Tambah control plane via Supabase Edge Function** untuk per-device config + key rotation
3. **Gantikan custom IPC** dengan abstraksi cross-platform `ITunnelSupervisor`

Hasil akhir: SASE yang **identity-aware**, **auto-rotate keys**, **per-user policy** (split tunnel), **roaming-aware**, dan **fully auditable** — tanpa rewrite WireGuard.

> Dokumen ini melengkapi [**Panduan TRMM Integration**](https://ardika.github.io/hermes-trmm-docs/) dengan pola arsitektur yang sama. Kalau Anda sudah membaca dokumen TRMM, banyak konsep di sini akan terasa familiar.

### Audiens

- **Engineer .NET / Avalonia** yang mengerjakan integrasi SASE di desktop client
- **Backend engineer** yang membangun Supabase Edge Function untuk config provisioning
- **DevOps / SRE** yang mengelola SASE gateway / WireGuard server
- **Security engineer** yang me-review threat model dan key management

### Prasyarat pengetahuan

- Pemrograman C# / .NET 8 (Avalonia)
- Konsep dasar VPN, routing, NAT
- Konsep dasar WireGuard (peer, public/private key, AllowedIPs)
- Operasi `wg`, `wg-quick`, `wireguard.exe` pada level dasar

---

## Daftar Isi

| # | Bab | Ringkasan |
|---|-----|-----------|
| 1 | [Pendahuluan]({{ site.baseurl }}{% link docs/01-pendahuluan.md %}) | Konteks, current state, masalah |
| 2 | [Arsitektur]({{ site.baseurl }}{% link docs/02-arsitektur.md %}) | Diagram before/after, tiga lapis |
| 3 | [Prasyarat]({{ site.baseurl }}{% link docs/03-prasyarat.md %}) | WireGuard tools, gateway, NuGet |
| 4 | [Layer 1 — TunnelSupervisor]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}) | Lifecycle WireGuard tunnel lokal |
| 5 | [Layer 2 — Config Service]({{ site.baseurl }}{% link docs/05-config-service.md %}) | Per-device config + Edge Function |
| 6 | [Layer 3 — Connection Flow]({{ site.baseurl }}{% link docs/06-connection-flow.md %}) | Bring-up, monitor, roaming, refresh |
| 7 | [Keamanan & Threat Model]({{ site.baseurl }}{% link docs/07-keamanan.md %}) | PSK, key rotation, identity-aware |
| 8 | [Dukungan macOS]({{ site.baseurl }}{% link docs/08-mac-support.md %}) | wg-quick, LaunchDaemon, signing |
| 9 | [WireGuard CLI Reference]({{ site.baseurl }}{% link docs/09-cli-reference.md %}) | Cheat-sheet `wg`, `wg-quick`, `wireguard.exe` |
| 10 | [Rencana Migrasi]({{ site.baseurl }}{% link docs/10-migrasi.md %}) | Phase 0–6, rollback |
| 11 | [Troubleshooting]({{ site.baseurl }}{% link docs/11-troubleshooting.md %}) | Error umum + debugging |
| 12 | [FAQ]({{ site.baseurl }}{% link docs/12-faq.md %}) | Pertanyaan & keputusan desain |

---

## Konvensi

- Kode contoh dalam C# ditarget untuk **.NET 8** (TFM `net8.0`).
- Kode TypeScript untuk Edge Function di-target Deno runtime.
- Path Windows: `C:\Path\To\File.exe`. macOS: `/Library/...`, `/etc/...`, `/usr/local/...`.
- Block `bash` = perintah shell di server / Mac.
- Block `powershell` = perintah di Windows.
- Block `csharp` = snippet untuk drop-in ke project Avalonia.

---

## Versi & Pembaruan

| Tanggal | Versi | Catatan |
|---|---|---|
| 2026-04-21 | 1.0 | Versi awal — refactor SASE arsitektur tiga-lapis dengan WireGuard data plane |
