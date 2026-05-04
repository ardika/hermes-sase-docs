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

`SaseConfigClient` adalah komponen di UI app yang **membaca konfigurasi WireGuard per-user** dari Supabase `user_data`. Karena Supabase yang dipakai self-hosted (tidak ada Edge Function), kita pakai **PostgREST query langsung** dengan RLS sebagai trust boundary.

**Prinsip:**

1. **Read-only dari client** untuk `wg_config` — populated by ops/admin di luar scope dokumen ini
2. **Write-only `wg_public_key` + `wg_last_handshake`** — client report-back untuk audit
3. **RLS enforce** semua akses — user hanya bisa lihat/edit row sendiri
4. **JWT user di header** — tiap request carry user identity, bukan API key privileged

## 5.2 Schema yang dipakai

Diasumsikan tabel `user_data` sudah ada di Supabase Anda. Kalau belum punya kolom yang dibutuhkan, jalankan migration:

```sql
ALTER TABLE user_data
  ADD COLUMN IF NOT EXISTS wg_config TEXT,
  ADD COLUMN IF NOT EXISTS wg_public_key TEXT,
  ADD COLUMN IF NOT EXISTS wg_last_handshake TIMESTAMPTZ,
  ADD COLUMN IF NOT EXISTS wg_status TEXT;

ALTER TABLE user_data ENABLE ROW LEVEL SECURITY;

CREATE POLICY IF NOT EXISTS "users_select_own"
  ON user_data FOR SELECT USING (auth.uid() = id);

CREATE POLICY IF NOT EXISTS "users_update_own_audit_fields"
  ON user_data FOR UPDATE USING (auth.uid() = id)
  WITH CHECK (auth.uid() = id);
```

Untuk **mencegah client mengubah `wg_config` sendiri** (yang harusnya read-only), tambah trigger:

```sql
CREATE OR REPLACE FUNCTION protect_wg_config()
RETURNS TRIGGER AS $$
BEGIN
  -- Only service_role bisa ubah wg_config
  IF NEW.wg_config IS DISTINCT FROM OLD.wg_config
     AND current_setting('request.jwt.claims', true)::json->>'role' != 'service_role' THEN
    RAISE EXCEPTION 'wg_config is managed by admin only';
  END IF;
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

DROP TRIGGER IF EXISTS trg_protect_wg_config ON user_data;
CREATE TRIGGER trg_protect_wg_config
  BEFORE UPDATE ON user_data FOR EACH ROW
  EXECUTE FUNCTION protect_wg_config();
```

### 5.2.1 Format `wg_config`

Dua opsi format yang umum dipakai:

**Format A: Full INI string (recommended — paling fleksibel)**

```
[Interface]
PrivateKey = <pre-generated-by-admin-or-placeholder>
Address = 10.99.0.42/32
DNS = 10.0.0.1, 1.1.1.1
MTU = 1280

[Peer]
PublicKey = <gateway-pub-base64>
PresharedKey = <psk-base64>
Endpoint = <gateway-host>:51820
AllowedIPs = 10.0.0.0/8, 192.168.0.0/16
PersistentKeepalive = 25
```

**Format B: JSON structured**

```json
{
  "interface": {
    "privateKey": "...",
    "address": "10.99.0.42/32",
    "dns": "10.0.0.1",
    "mtu": 1280
  },
  "peers": [{
    "publicKey": "...",
    "presharedKey": "...",
    "endpoint": "...",
    "allowedIPs": "10.0.0.0/8",
    "persistentKeepalive": 25
  }]
}
```

Dokumen ini support kedua-duanya — parser di client deteksi format dari content.

> **Catatan keamanan tentang PrivateKey di `wg_config`:** kalau private key WireGuard **disimpan langsung** di `wg_config` di Supabase, itu menerima trust model "admin pegang semua keys". Acceptable kalau admin = orang yang sama yang manage user, tapi sub-optimal untuk threat model production.
>
> **Pattern yang lebih aman**: client generate keypair sendiri (`KeyStore`), publish hanya pubkey ke `user_data.wg_public_key`, admin (lewat tooling/automation) generate peer registration di gateway dan return WG config tanpa PrivateKey ke `user_data.wg_config`. Client inject local PrivateKey saat apply ke Helper. Lihat §5.5 untuk implementasi.

## 5.3 DTO

```csharp
namespace HermesNetwork.Sase.Config.Models;

public sealed record SaseConfigDto(
    string RawConfig,                        // INI string lengkap setelah inject PrivateKey
    DateTimeOffset? LastUpdated,             // dari user_data.updated_at
    string? Status                            // user_data.wg_status (kalau ada)
);

public sealed record UserDataRow(
    string Id,
    string? WgConfig,                        // INI atau JSON, nullable kalau belum ada
    string? WgPublicKey,
    DateTimeOffset? UpdatedAt
);
```

## 5.4 KeyStore (per-device WireGuard keypair)

Sama seperti pattern di TRMM docs — keypair di-generate sekali per device, simpan dengan DPAPI/Keychain.

```csharp
using System.Diagnostics;
using System.IO;
using System.Security.Cryptography;
using System.Text.Json;

namespace HermesNetwork.Sase.Config;

public sealed class KeyStore
{
    private readonly string _path;

    public KeyStore()
    {
        var dir = OperatingSystem.IsWindows()
            ? Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
                           "HermesNetwork360Guard", "Sase")
            : Path.Combine(Environment.GetFolderPath(Environment.SpecialFolder.UserProfile),
                           "Library", "Application Support", "HermesNetwork360Guard", "Sase");
        Directory.CreateDirectory(dir);
        _path = Path.Combine(dir, "keypair.json");
    }

    public async Task<KeyPair> GetOrCreateAsync()
    {
        var existing = await LoadAsync();
        if (existing is not null) return existing;

        var (priv, pub) = await GenerateAsync();
        var pair = new KeyPair(priv, pub, DateTimeOffset.UtcNow);
        await SaveAsync(pair);
        return pair;
    }

    private async Task<KeyPair?> LoadAsync()
    {
        if (!File.Exists(_path)) return null;
        var data = await File.ReadAllBytesAsync(_path);

        if (OperatingSystem.IsWindows())
        {
            var clear = ProtectedData.Unprotect(data, null, DataProtectionScope.CurrentUser);
            return JsonSerializer.Deserialize<KeyPair>(clear);
        }
        return JsonSerializer.Deserialize<KeyPair>(data);
    }

    private async Task SaveAsync(KeyPair pair)
    {
        var json = JsonSerializer.SerializeToUtf8Bytes(pair);
        if (OperatingSystem.IsWindows())
        {
            var enc = ProtectedData.Protect(json, null, DataProtectionScope.CurrentUser);
            await File.WriteAllBytesAsync(_path, enc);
        }
        else
        {
            await File.WriteAllBytesAsync(_path, json);
            File.SetUnixFileMode(_path, UnixFileMode.UserRead | UnixFileMode.UserWrite);
        }
    }

    private static async Task<(string Priv, string Pub)> GenerateAsync()
    {
        var wg = OperatingSystem.IsWindows()
            ? @"C:\Program Files\WireGuard\wg.exe"
            : "/usr/local/bin/wg";

        var priv = (await RunAsync(wg, "genkey")).Trim();
        var pub  = (await RunWithStdinAsync(wg, "pubkey", priv)).Trim();
        return (priv, pub);
    }

    private static async Task<string> RunAsync(string file, string args)
    {
        var psi = new ProcessStartInfo(file, args)
        {
            RedirectStandardOutput = true, UseShellExecute = false, CreateNoWindow = true,
        };
        using var p = Process.Start(psi)!;
        var stdout = await p.StandardOutput.ReadToEndAsync();
        await p.WaitForExitAsync();
        return stdout;
    }

    private static async Task<string> RunWithStdinAsync(string file, string args, string stdin)
    {
        var psi = new ProcessStartInfo(file, args)
        {
            RedirectStandardInput = true, RedirectStandardOutput = true,
            UseShellExecute = false, CreateNoWindow = true,
        };
        using var p = Process.Start(psi)!;
        await p.StandardInput.WriteAsync(stdin);
        p.StandardInput.Close();
        var stdout = await p.StandardOutput.ReadToEndAsync();
        await p.WaitForExitAsync();
        return stdout;
    }

    public sealed record KeyPair(string PrivateKey, string PublicKey, DateTimeOffset GeneratedAt);
}
```

## 5.5 SaseConfigClient

```csharp
using System.Net.Http;
using System.Net.Http.Json;
using System.Text.Json.Serialization;

namespace HermesNetwork.Sase.Config;

public interface ISaseConfigClient
{
    Task<string> GetConfigAsync(CancellationToken ct = default);
    Task ReportStatusAsync(string status, DateTimeOffset? lastHandshake, CancellationToken ct = default);
}

public sealed class SaseConfigClient : ISaseConfigClient
{
    private readonly HttpClient _http;
    private readonly KeyStore _keyStore;
    private readonly Func<string> _userJwtProvider;
    private readonly string _supabaseUrl;
    private readonly string _anonKey;

    public SaseConfigClient(
        HttpClient http,
        KeyStore keyStore,
        Func<string> userJwtProvider,
        string supabaseUrl,
        string anonKey)
    {
        _http = http;
        _keyStore = keyStore;
        _userJwtProvider = userJwtProvider;
        _supabaseUrl = supabaseUrl.TrimEnd('/');
        _anonKey = anonKey;
    }

    public async Task<string> GetConfigAsync(CancellationToken ct = default)
    {
        // 1. Read user_data via PostgREST (RLS filter ke row user)
        var req = new HttpRequestMessage(HttpMethod.Get,
            $"{_supabaseUrl}/rest/v1/user_data?select=id,wg_config,wg_public_key&limit=1");
        req.Headers.Add("apikey", _anonKey);
        req.Headers.Add("Authorization", $"Bearer {_userJwtProvider()}");

        using var resp = await _http.SendAsync(req, ct);
        resp.EnsureSuccessStatusCode();

        var rows = await resp.Content.ReadFromJsonAsync<UserDataRow[]>(cancellationToken: ct);
        if (rows is null || rows.Length == 0)
            throw new InvalidOperationException("user_data row not found for current user");
        var row = rows[0];

        if (string.IsNullOrWhiteSpace(row.WgConfig))
            throw new InvalidOperationException(
                "wg_config kosong di user_data. Hubungi admin untuk provisioning.");

        // 2. Get / create local keypair
        var keypair = await _keyStore.GetOrCreateAsync();

        // 3. Kalau public key di-server beda dengan keypair lokal → write back
        if (row.WgPublicKey != keypair.PublicKey)
        {
            await UpdatePublicKeyAsync(row.Id, keypair.PublicKey, ct);
        }

        // 4. Parse + inject PrivateKey lokal
        var configIni = NormalizeToIni(row.WgConfig);
        var withPrivateKey = InjectPrivateKey(configIni, keypair.PrivateKey);
        return withPrivateKey;
    }

    public async Task ReportStatusAsync(
        string status, DateTimeOffset? lastHandshake, CancellationToken ct = default)
    {
        var jwt = _userJwtProvider();
        // Cara cepat: PATCH user_data row sendiri (RLS allow)
        var req = new HttpRequestMessage(HttpMethod.Patch,
            $"{_supabaseUrl}/rest/v1/user_data?id=eq.{await GetUserIdFromJwt(ct)}")
        {
            Content = JsonContent.Create(new
            {
                wg_status = status,
                wg_last_handshake = lastHandshake
            })
        };
        req.Headers.Add("apikey", _anonKey);
        req.Headers.Add("Authorization", $"Bearer {jwt}");
        req.Headers.Add("Prefer", "return=minimal");
        using var resp = await _http.SendAsync(req, ct);
        resp.EnsureSuccessStatusCode();
    }

    private async Task UpdatePublicKeyAsync(string userId, string pubkey, CancellationToken ct)
    {
        var req = new HttpRequestMessage(HttpMethod.Patch,
            $"{_supabaseUrl}/rest/v1/user_data?id=eq.{userId}")
        {
            Content = JsonContent.Create(new { wg_public_key = pubkey })
        };
        req.Headers.Add("apikey", _anonKey);
        req.Headers.Add("Authorization", $"Bearer {_userJwtProvider()}");
        req.Headers.Add("Prefer", "return=minimal");
        using var resp = await _http.SendAsync(req, ct);
        resp.EnsureSuccessStatusCode();
    }

    private async Task<string> GetUserIdFromJwt(CancellationToken ct)
    {
        // Decode JWT payload (tidak verify; hanya read sub claim)
        var jwt = _userJwtProvider();
        var parts = jwt.Split('.');
        if (parts.Length < 2) throw new InvalidOperationException("Invalid JWT");
        var payload = parts[1].Replace('-', '+').Replace('_', '/');
        switch (payload.Length % 4) { case 2: payload += "=="; break; case 3: payload += "="; break; }
        var json = System.Text.Encoding.UTF8.GetString(Convert.FromBase64String(payload));
        using var doc = System.Text.Json.JsonDocument.Parse(json);
        return doc.RootElement.GetProperty("sub").GetString()!;
    }

    /// <summary>Convert format JSON → INI kalau perlu, atau return as-is kalau sudah INI.</summary>
    private static string NormalizeToIni(string raw)
    {
        var trimmed = raw.TrimStart();
        if (trimmed.StartsWith("[Interface]")) return raw;   // already INI
        if (trimmed.StartsWith("{"))
        {
            // JSON → INI
            using var doc = System.Text.Json.JsonDocument.Parse(raw);
            return JsonToIni(doc.RootElement);
        }
        throw new InvalidOperationException("Unknown wg_config format");
    }

    private static string JsonToIni(System.Text.Json.JsonElement root)
    {
        var sb = new System.Text.StringBuilder();
        var iface = root.GetProperty("interface");
        sb.AppendLine("[Interface]");
        sb.AppendLine($"Address = {iface.GetProperty("address").GetString()}");
        if (iface.TryGetProperty("dns", out var dns))
            sb.AppendLine($"DNS = {dns.GetString()}");
        if (iface.TryGetProperty("mtu", out var mtu))
            sb.AppendLine($"MTU = {mtu.GetInt32()}");
        sb.AppendLine();

        foreach (var p in root.GetProperty("peers").EnumerateArray())
        {
            sb.AppendLine("[Peer]");
            sb.AppendLine($"PublicKey = {p.GetProperty("publicKey").GetString()}");
            if (p.TryGetProperty("presharedKey", out var psk))
                sb.AppendLine($"PresharedKey = {psk.GetString()}");
            sb.AppendLine($"Endpoint = {p.GetProperty("endpoint").GetString()}");
            sb.AppendLine($"AllowedIPs = {p.GetProperty("allowedIPs").GetString()}");
            if (p.TryGetProperty("persistentKeepalive", out var ka))
                sb.AppendLine($"PersistentKeepalive = {ka.GetInt32()}");
            sb.AppendLine();
        }
        return sb.ToString();
    }

    /// <summary>
    /// Sisipkan PrivateKey ke section [Interface]. Kalau sudah ada PrivateKey
    /// (mis. admin pre-populate), replace dengan local key.
    /// </summary>
    private static string InjectPrivateKey(string ini, string privateKey)
    {
        var lines = ini.Split('\n').ToList();
        var ifaceIdx = lines.FindIndex(l => l.TrimStart().StartsWith("[Interface]"));
        if (ifaceIdx < 0)
            throw new InvalidOperationException("No [Interface] section");

        var existingPkIdx = -1;
        for (int i = ifaceIdx + 1; i < lines.Count && !lines[i].TrimStart().StartsWith("["); i++)
        {
            if (lines[i].TrimStart().StartsWith("PrivateKey"))
            {
                existingPkIdx = i; break;
            }
        }
        var newLine = $"PrivateKey = {privateKey}";
        if (existingPkIdx >= 0) lines[existingPkIdx] = newLine;
        else lines.Insert(ifaceIdx + 1, newLine);

        return string.Join("\n", lines);
    }

    private sealed record UserDataRow(
        [property: JsonPropertyName("id")]            string Id,
        [property: JsonPropertyName("wg_config")]     string? WgConfig,
        [property: JsonPropertyName("wg_public_key")] string? WgPublicKey
    );
}
```

## 5.6 Subscribe Realtime (opsional)

Kalau Realtime aktif di Supabase self-hosted, subscribe ke `user_data` row sendiri untuk dapat push update saat admin ubah `wg_config`:

```csharp
using Supabase.Realtime;

public sealed class SaseConfigRealtimeListener : IAsyncDisposable
{
    private readonly Supabase.Client _supabase;
    private RealtimeChannel? _channel;

    public event Action? ConfigUpdated;

    public SaseConfigRealtimeListener(Supabase.Client supabase) { _supabase = supabase; }

    public async Task StartAsync(string userId)
    {
        _channel = _supabase.Realtime.Channel("realtime", "public", "user_data",
            filter: $"id=eq.{userId}");
        _channel.AddPostgresChangeHandler(
            Supabase.Realtime.PostgresChanges.PostgresChangesOptions.ListenType.Updates,
            (_, change) => ConfigUpdated?.Invoke());
        await _channel.Subscribe();
    }

    public async ValueTask DisposeAsync()
    {
        if (_channel is not null) await _channel.Unsubscribe();
    }
}
```

Fallback kalau Realtime tidak available: polling tiap 5 menit query `wg_config` lalu compare.

## 5.7 Setup di Avalonia DI

```csharp
// di Program.cs / App startup
services.AddHttpClient<ISaseConfigClient, SaseConfigClient>();
services.AddSingleton<KeyStore>();

services.AddSingleton<ISaseConfigClient>(sp =>
    new SaseConfigClient(
        sp.GetRequiredService<IHttpClientFactory>().CreateClient(),
        sp.GetRequiredService<KeyStore>(),
        () => sp.GetRequiredService<SupabaseSession>().AccessToken!,
        Configuration["HNGUARD_SUPABASE_URL"]!,
        Configuration["HNGUARD_SUPABASE_ANON_KEY"]!));
```

## 5.8 Test integration

```bash
# 1. Login user test, dapat JWT
JWT=$(curl -X POST "$SB_URL/auth/v1/token?grant_type=password" \
  -H "apikey: $ANON" -H "Content-Type: application/json" \
  -d '{"email":"test@hermes","password":"..."}' | jq -r .access_token)

# 2. Read user_data
curl "$SB_URL/rest/v1/user_data?select=*" \
  -H "apikey: $ANON" -H "Authorization: Bearer $JWT" | jq

# 3. Test parser di .NET
dotnet run --project HermesNetwork.SaseConfigTest
# (test program manual yang panggil GetConfigAsync, print output)
```

## 5.9 Best practices

- **Cache config di memory** setelah read pertama; refresh on-demand atau via Realtime, bukan tiap operasi
- **Inject PrivateKey paling akhir** — pastikan key tidak masuk ke log apa pun
- **Sanitize log message** kalau dump config — redact `PrivateKey`, `PresharedKey` lines
- **Handle network error gracefully** — kalau Supabase tidak reachable, pakai cached config terakhir kalau ada
- **Validate INI sebelum kirim ke Helper** — lebih baik fail early daripada Helper reject

---

[← Bab 4 Helper Service]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}){: .btn }
[Bab 6 — Connection Flow →]({{ site.baseurl }}{% link docs/06-connection-flow.md %}){: .btn .btn-primary }
