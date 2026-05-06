---
layout: default
title: 5. Config Service (Supabase)
nav_order: 6
permalink: /docs/config-service/
---

# 5. Config Service — Read dari Supabase user_data
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 5.1 Tujuan

`SaseConfigClient` adalah komponen di UI app yang **membaca konfigurasi WireGuard per-user** dari Supabase `user_data.configuration`. Karena Supabase yang dipakai self-hosted (tidak ada Edge Function), kita pakai **PostgREST query langsung** dengan RLS.

> **PENTING:** Schema Supabase **TIDAK BOLEH DIUBAH**. Kita pakai kolom yang sudah ada di production:
> - `user_data.configuration` (text) — full WireGuard INI string
> - `user_data.assigned_ip`, `sase_slice_id`, `sase_version` (informational)
> - RLS yang sudah aktif (`uid() = uid`) — auto-filter ke row user yang login

**Prinsip:**
1. **Read-only dari client** untuk semua kolom — admin tooling yang populate `configuration`.
2. **Tidak generate keypair di client** — admin sudah pre-generate dan stuff `PrivateKey` di dalam `configuration`.
3. **Tidak ada write-back** dari client ke `user_data` — disiplin di kode (RLS sebenarnya allow, tapi kita tidak pakai).
4. **JWT user di header** — RLS otomatis filter ke row user.

## 5.2 Realitas schema production

Schema yang dipakai (sudah ada, tidak perlu migration):

```sql
-- Sudah ada di production. JANGAN dibuat ulang.
-- Tabel user_data dengan 23 kolom; yang relevan:
--
--   uid           uuid          -- linked ke auth user via uid() = uid
--   configuration text          -- full WG INI string (dari admin tooling)
--   assigned_ip   text          -- IP user di tunnel (informational)
--   sase_slice_id int4          -- slice/network di gateway
--   sase_version  text          -- (kosong di production saat ini)
--
-- RLS sudah aktif dengan policy "Allow all usage":
--   USING (uid() = uid) WITH CHECK (uid() = uid)
```

### 5.2.1 Format `configuration` yang ditemukan di production

Berdasarkan inspeksi 5 sampel real user, format `configuration` adalah:

- **INI string WireGuard standar**, length ~220–250 byte
- **`[Interface]` section di posisi 1** — selalu di awal
- **Mengandung:** `PrivateKey`, `Address`, `DNS`, `[Peer]`, `PublicKey`, `Endpoint`, `AllowedIPs`
- **TIDAK mengandung:** `PresharedKey`, `MTU`, `PersistentKeepalive`

Contoh struktur (nilai diredact):

```ini
[Interface]
PrivateKey = <base64-private-key-yang-pre-generated-admin>
Address = 10.x.x.x/32
DNS = 10.x.x.x

[Peer]
PublicKey = <base64-gateway-public-key>
Endpoint = <gateway-host>:51820
AllowedIPs = 0.0.0.0/0
```

Implikasi penting:
- **Client tidak generate keypair sendiri** — admin yang manage. PrivateKey datang dari Supabase.
- **Tidak ada PSK** di production — jangan asumsikan ada di kode parser/builder.
- **Tidak ada keepalive** — kalau perlu NAT traversal, bicarakan dengan ops untuk tambah di config admin tooling, bukan di client.

> Karena PrivateKey **disimpan di Supabase**, threat model harus reflect ini. Lihat [Bab 7 — Keamanan]({{ site.baseurl }}{% link docs/07-keamanan.md %}) §7.4. Jangan duplicate PrivateKey ke disk client lebih lama dari yang dibutuhkan.

## 5.3 DTO

```csharp
namespace HermesNetwork.Sase.Config.Models;

/// <summary>Hasil read dari user_data.</summary>
public sealed record SaseConfigDto(
    string Configuration,             // full INI string apa adanya
    string? AssignedIp,
    int? SaseSliceId,
    string? SaseVersion);
```

Tidak ada `PrivateKey`, `Peers`, atau struktur parsed — kita **passthrough INI string apa adanya** ke Helper. Helper yang akan tulis ke disk WireGuard.

## 5.4 SaseConfigClient (full implementation)

```csharp
using System.Net.Http;
using System.Net.Http.Json;
using System.Text.Json.Serialization;
using HermesNetwork.Sase.Config.Models;

namespace HermesNetwork.Sase.Config;

public interface ISaseConfigClient
{
    /// <summary>Return raw INI WireGuard config dari user_data.configuration.</summary>
    Task<string> GetConfigAsync(CancellationToken ct = default);

    /// <summary>Return DTO dengan kolom tambahan untuk telemetry/debug.</summary>
    Task<SaseConfigDto> GetFullAsync(CancellationToken ct = default);
}

public sealed class SaseConfigClient : ISaseConfigClient
{
    private readonly HttpClient _http;
    private readonly Func<string> _userJwtProvider;
    private readonly string _supabaseUrl;
    private readonly string _anonKey;

    public SaseConfigClient(
        HttpClient http,
        Func<string> userJwtProvider,
        string supabaseUrl,
        string anonKey)
    {
        _http = http;
        _userJwtProvider = userJwtProvider;
        _supabaseUrl = supabaseUrl.TrimEnd('/');
        _anonKey = anonKey;
    }

    public async Task<string> GetConfigAsync(CancellationToken ct = default)
    {
        var dto = await GetFullAsync(ct);
        return dto.Configuration;
    }

    public async Task<SaseConfigDto> GetFullAsync(CancellationToken ct = default)
    {
        // RLS auto-filter ke row user dari JWT, jadi tidak perlu WHERE id=...
        var url = $"{_supabaseUrl}/rest/v1/user_data" +
                  "?select=configuration,assigned_ip,sase_slice_id,sase_version" +
                  "&limit=1";

        using var req = new HttpRequestMessage(HttpMethod.Get, url);
        req.Headers.Add("apikey", _anonKey);
        req.Headers.Add("Authorization", $"Bearer {_userJwtProvider()}");

        using var resp = await _http.SendAsync(req, ct);
        resp.EnsureSuccessStatusCode();

        var rows = await resp.Content.ReadFromJsonAsync<UserDataRow[]>(cancellationToken: ct);
        if (rows is null || rows.Length == 0)
            throw new InvalidOperationException(
                "user_data row not found untuk current user. " +
                "Pastikan user sudah login dan punya row di Supabase.");

        var row = rows[0];
        if (string.IsNullOrWhiteSpace(row.Configuration))
            throw new InvalidOperationException(
                "user_data.configuration kosong. " +
                "Hubungi admin untuk provisioning WG config.");

        // Sanity check: pastikan minimal ada [Interface] dan [Peer]
        if (!row.Configuration.Contains("[Interface]") ||
            !row.Configuration.Contains("[Peer]"))
            throw new InvalidOperationException(
                "user_data.configuration tidak terlihat seperti WireGuard INI. " +
                $"Length={row.Configuration.Length}, prefix=\"{row.Configuration[..Math.Min(40, row.Configuration.Length)]}\"");

        return new SaseConfigDto(
            Configuration:  row.Configuration,
            AssignedIp:     row.AssignedIp,
            SaseSliceId:    row.SaseSliceId,
            SaseVersion:    row.SaseVersion);
    }

    private sealed record UserDataRow(
        [property: JsonPropertyName("configuration")]   string? Configuration,
        [property: JsonPropertyName("assigned_ip")]     string? AssignedIp,
        [property: JsonPropertyName("sase_slice_id")]   int? SaseSliceId,
        [property: JsonPropertyName("sase_version")]    string? SaseVersion
    );
}
```

**Yang dihapus dari versi sebelumnya:**
- ❌ `KeyStore` class — tidak generate keypair di client
- ❌ `KeyPairGenerator` — tidak panggil `wg genkey`
- ❌ `DeviceIdHelper` — tidak butuh device ID untuk write ke `wg_public_key` (kolomnya pun tidak ada)
- ❌ `ConfigRequestDto` — tidak post ke Edge Function
- ❌ `JsonToIni()` parser — admin selalu populate INI, bukan JSON
- ❌ `InjectPrivateKey()` — admin sudah include PrivateKey di `configuration`
- ❌ `UpdatePublicKeyAsync()` — tidak write back ke Supabase
- ❌ `ReportStatusAsync()` — kolom `wg_status` / `wg_last_handshake` tidak ada; gunakan app log lokal saja
- ❌ Interaksi dengan Edge Function — tidak ada Edge Function di self-hosted

## 5.5 Polling refresh (Realtime tidak available)

Karena Realtime di self-hosted Supabase tidak aktif, gunakan **polling tiap 5 menit** di `ConnectionService` background monitor untuk detect perubahan config:

```csharp
// di SaseConnectionService.MonitorLoopAsync (lihat Bab 6)
var lastConfigCheck = DateTime.UtcNow;
var lastConfigHash = ComputeHash(currentConfig);

while (!ct.IsCancellationRequested)
{
    await Task.Delay(TimeSpan.FromSeconds(20), ct);

    // ... status check existing ...

    if (DateTime.UtcNow - lastConfigCheck > TimeSpan.FromMinutes(5))
    {
        lastConfigCheck = DateTime.UtcNow;
        try
        {
            var fresh = await _config.GetConfigAsync(ct);
            var freshHash = ComputeHash(fresh);
            if (freshHash != lastConfigHash)
            {
                _log.LogInformation("Config changed, applying new config");
                await _helper.ApplyConfigAsync(TunnelName, fresh, ct);
                await _helper.StartTunnelAsync(TunnelName, ct);   // restart utk apply
                lastConfigHash = freshHash;
            }
        }
        catch (Exception ex)
        {
            _log.LogWarning(ex, "Config refresh check failed");
        }
    }
}

static string ComputeHash(string s) =>
    Convert.ToBase64String(System.Security.Cryptography.SHA256.HashData(
        System.Text.Encoding.UTF8.GetBytes(s)));
```

## 5.6 Setup di Avalonia DI

```csharp
// di Program.cs / App startup
services.AddHttpClient();
services.AddSingleton<ISaseConfigClient>(sp =>
    new SaseConfigClient(
        sp.GetRequiredService<IHttpClientFactory>().CreateClient(),
        () => sp.GetRequiredService<SupabaseSession>().AccessToken!,
        Configuration["HNGUARD_SUPABASE_URL"]!,
        Configuration["HNGUARD_SUPABASE_ANON_KEY"]!));
```

`SupabaseSession` adalah kelas existing di Hermes (atau wrapper minimal) yang expose `AccessToken` JWT user yang sedang login.

## 5.7 Test integration

```bash
# 1. Login user test, dapat JWT
JWT=$(curl -X POST "$SB_URL/auth/v1/token?grant_type=password" \
  -H "apikey: $ANON" -H "Content-Type: application/json" \
  -d '{"email":"test@hermes","password":"..."}' | jq -r .access_token)

# 2. Read user_data — RLS otomatis filter ke row user
curl "$SB_URL/rest/v1/user_data?select=configuration,assigned_ip,sase_slice_id,sase_version&limit=1" \
  -H "apikey: $ANON" -H "Authorization: Bearer $JWT" | jq
# Expected:
# [{
#   "configuration": "[Interface]\nPrivateKey = ...\nAddress = ...\nDNS = ...\n[Peer]\nPublicKey = ...\nEndpoint = ...\nAllowedIPs = ...\n",
#   "assigned_ip": "...",
#   "sase_slice_id": 21,
#   "sase_version": ""
# }]
```

```csharp
// 3. Test di .NET
var dto = await _configClient.GetFullAsync();
dto.Configuration.Should().Contain("[Interface]");
dto.Configuration.Should().Contain("[Peer]");
dto.Configuration.Should().Contain("PrivateKey");
dto.Configuration.Should().Contain("PublicKey");
dto.Configuration.Should().Contain("Endpoint");
dto.Configuration.Should().Contain("AllowedIPs");
// PSK / MTU / Keepalive optional — jangan assert harus ada
```

## 5.8 Best practices

- **Cache config di memory** setelah read pertama (di `SaseConnectionService`); refresh on-demand atau via polling 5 menit, bukan tiap operasi.
- **Hash config (SHA-256)** untuk detect perubahan tanpa harus banding string panjang.
- **Sanitize log message** — JANGAN log `configuration` content (mengandung PrivateKey).
- **Handle network error gracefully** — kalau Supabase tidak reachable, pakai cached config terakhir kalau ada.
- **Validate INI minimal** — `[Interface]` + `[Peer]` ada — sebelum kirim ke Helper.
- **Treat config sebagai opaque blob** dari sudut pandang client — admin yang berhak ubah, client cuma baca + apply.
- **Jangan tulis fallback default values** untuk PSK/MTU/Keepalive di kode — kalau admin populate, akan otomatis terapply; kalau tidak, biarkan WireGuard pakai default.

---

[← Bab 4 Helper Service]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}){: .btn }
[Bab 6 — Connection Flow →]({{ site.baseurl }}{% link docs/06-connection-flow.md %}){: .btn .btn-primary }
