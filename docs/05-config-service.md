---
layout: default
title: 5. Layer 2 — Config Service
nav_order: 6
permalink: /docs/config-service/
---

# 5. Layer 2 — Config Service
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 5.1 Tujuan

`SaseConfigClient` + Edge Function `sase-config` adalah **trust boundary** antara desktop client dan SASE control plane:

- Client minta config via Edge Function dengan user JWT
- Edge Function memverifikasi JWT, ambil role user, register peer di gateway pakai API key, dan return config
- Client tidak pernah lihat API key SASE

**Prinsip utama:**

1. **Private key tidak meninggalkan client** — generate lokal, kirim hanya public key
2. **API key SASE tidak embed di client** — hanya di Supabase Edge Function
3. **Per-device config** — tidak shared antar device walaupun user sama
4. **Per-role policy** — AllowedIPs / DNS / split-tunnel ditentukan oleh role user di Supabase

## 5.2 Schema database (Supabase)

Tambahkan kolom & tabel:

```sql
-- Kolom di user_profiles untuk role mapping
ALTER TABLE user_profiles
  ADD COLUMN IF NOT EXISTS sase_role TEXT DEFAULT 'engineer';
  -- nilai: 'engineer' | 'executive' | 'intern' | 'contractor'

-- Tabel registrasi peer
CREATE TABLE IF NOT EXISTS sase_peer (
    id              BIGSERIAL PRIMARY KEY,
    user_id         UUID NOT NULL REFERENCES auth.users(id),
    device_id       TEXT NOT NULL,                 -- hostname + machine GUID
    public_key      TEXT NOT NULL UNIQUE,          -- WireGuard pub key (b64)
    psk             TEXT NOT NULL,                 -- pre-shared key (b64)
    interface_ip    TEXT NOT NULL,                 -- IP yang di-assign di tunnel
    role_at_issue   TEXT NOT NULL,
    issued_at       TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    last_refreshed  TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    expires_at      TIMESTAMPTZ NOT NULL,           -- TTL config (mis. +24 jam)
    status          TEXT NOT NULL DEFAULT 'active'  -- 'active' | 'revoked' | 'expired'
);

CREATE UNIQUE INDEX idx_sase_peer_user_device ON sase_peer (user_id, device_id);
CREATE INDEX idx_sase_peer_pubkey ON sase_peer (public_key);

ALTER TABLE sase_peer ENABLE ROW LEVEL SECURITY;
CREATE POLICY "users_own_sase_peers" ON sase_peer
  FOR SELECT USING (auth.uid() = user_id);

-- Audit log
CREATE TABLE IF NOT EXISTS sase_audit_log (
    id           BIGSERIAL PRIMARY KEY,
    user_id      UUID NOT NULL,
    device_id    TEXT,
    action       TEXT NOT NULL,            -- 'config_request' | 'config_refresh' | 'revoke'
    details      JSONB,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

## 5.3 DTO models di .NET

### `SaseConfigDto.cs`

```csharp
using System.Text.Json.Serialization;

namespace HermesNetwork.Sase.Config.Models;

public sealed record SaseConfigDto(
    [property: JsonPropertyName("private_key")]      string PrivateKey,
    [property: JsonPropertyName("interface_address")] string InterfaceAddress,
    [property: JsonPropertyName("dns")]              string? Dns,
    [property: JsonPropertyName("mtu")]              int? Mtu,
    [property: JsonPropertyName("peers")]            PeerDto[] Peers,
    [property: JsonPropertyName("expires_at")]       DateTimeOffset ExpiresAt
);

public sealed record PeerDto(
    [property: JsonPropertyName("public_key")]   string PublicKey,
    [property: JsonPropertyName("preshared_key")] string? PresharedKey,
    [property: JsonPropertyName("endpoint")]     string Endpoint,
    [property: JsonPropertyName("allowed_ips")]  string AllowedIPs,
    [property: JsonPropertyName("keepalive_seconds")] int KeepaliveSeconds
);

public sealed record ConfigRequestDto(
    [property: JsonPropertyName("device_id")]   string DeviceId,
    [property: JsonPropertyName("hostname")]    string Hostname,
    [property: JsonPropertyName("public_key")]  string PublicKey,
    [property: JsonPropertyName("platform")]    string Platform   // "windows" | "darwin"
);
```

## 5.4 KeyPairGenerator

WireGuard punya CLI `wg genkey` & `wg pubkey`. Cara paling reliable: panggil binary tersebut (sudah terinstall sebagai prasyarat).

```csharp
using System.Diagnostics;
using System.Threading.Tasks;

namespace HermesNetwork.Sase.Config;

public static class KeyPairGenerator
{
    public static async Task<(string PrivateKey, string PublicKey)> GenerateAsync()
    {
        var wgPath = OperatingSystem.IsWindows()
            ? @"C:\Program Files\WireGuard\wg.exe"
            : "/usr/local/bin/wg";   // Mac

        // wg genkey -> base64 private key
        var priv = (await RunAsync(wgPath, "genkey")).Stdout.Trim();

        // wg pubkey, ambil dari stdin
        var pub = await RunWithStdinAsync(wgPath, "pubkey", priv);

        return (priv, pub.Trim());
    }

    private static async Task<(string Stdout, string Stderr, int Code)> RunAsync(
        string file, string args)
    {
        var psi = new ProcessStartInfo(file, args)
        {
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            CreateNoWindow = true,
        };
        using var p = Process.Start(psi)!;
        var so = p.StandardOutput.ReadToEndAsync();
        var se = p.StandardError.ReadToEndAsync();
        await p.WaitForExitAsync();
        return (await so, await se, p.ExitCode);
    }

    private static async Task<string> RunWithStdinAsync(
        string file, string args, string stdin)
    {
        var psi = new ProcessStartInfo(file, args)
        {
            RedirectStandardInput = true,
            RedirectStandardOutput = true,
            UseShellExecute = false,
            CreateNoWindow = true,
        };
        using var p = Process.Start(psi)!;
        await p.StandardInput.WriteAsync(stdin);
        p.StandardInput.Close();
        var stdout = await p.StandardOutput.ReadToEndAsync();
        await p.WaitForExitAsync();
        return stdout;
    }
}
```

## 5.5 KeyStore (cache keypair lokal)

Keypair perlu disimpan **sekali per device** dan di-reuse. Kalau hilang, kita generate baru dan re-register.

```csharp
using System;
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

    public async Task<KeyPair?> LoadAsync()
    {
        if (!File.Exists(_path)) return null;
        var data = await File.ReadAllBytesAsync(_path);

        if (OperatingSystem.IsWindows())
        {
            // DPAPI unprotect
            var clear = ProtectedData.Unprotect(data, null, DataProtectionScope.CurrentUser);
            return JsonSerializer.Deserialize<KeyPair>(clear);
        }
        else
        {
            // Mac: file sudah 0600 di home dir; bisa enhance dengan Keychain di phase berikutnya
            return JsonSerializer.Deserialize<KeyPair>(data);
        }
    }

    public async Task SaveAsync(KeyPair pair)
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
            // Set 0600
            File.SetUnixFileMode(_path,
                UnixFileMode.UserRead | UnixFileMode.UserWrite);
        }
    }

    public sealed record KeyPair(string PrivateKey, string PublicKey, DateTimeOffset GeneratedAt);
}
```

## 5.6 SaseConfigClient

```csharp
using System.Net.Http;
using System.Net.Http.Json;
using HermesNetwork.Sase.Config.Models;

namespace HermesNetwork.Sase.Config;

public interface ISaseConfigClient
{
    Task<SaseConfigDto> RequestConfigAsync(CancellationToken ct = default);
    Task<bool> IsConfigFreshAsync(CancellationToken ct = default);
}

public sealed class SaseConfigClient : ISaseConfigClient
{
    private readonly HttpClient _http;
    private readonly KeyStore _keyStore;
    private readonly Func<string> _userJwtProvider;
    private readonly string _supabaseUrl;

    public SaseConfigClient(
        HttpClient http,
        KeyStore keyStore,
        Func<string> userJwtProvider,
        string supabaseUrl)
    {
        _http = http;
        _keyStore = keyStore;
        _userJwtProvider = userJwtProvider;
        _supabaseUrl = supabaseUrl.TrimEnd('/');
    }

    public async Task<SaseConfigDto> RequestConfigAsync(CancellationToken ct = default)
    {
        // 1. Load atau generate keypair
        var pair = await _keyStore.LoadAsync();
        if (pair is null)
        {
            var (priv, pub) = await KeyPairGenerator.GenerateAsync();
            pair = new KeyStore.KeyPair(priv, pub, DateTimeOffset.UtcNow);
            await _keyStore.SaveAsync(pair);
        }

        // 2. POST ke edge function
        var hostname = Environment.MachineName;
        var deviceId = await DeviceIdHelper.GetAsync();
        var platform = OperatingSystem.IsWindows() ? "windows" : "darwin";

        var req = new HttpRequestMessage(HttpMethod.Post,
            $"{_supabaseUrl}/functions/v1/sase-config")
        {
            Content = JsonContent.Create(new ConfigRequestDto(
                DeviceId:  deviceId,
                Hostname:  hostname,
                PublicKey: pair.PublicKey,
                Platform:  platform))
        };
        req.Headers.Add("Authorization", $"Bearer {_userJwtProvider()}");

        using var resp = await _http.SendAsync(req, ct);
        resp.EnsureSuccessStatusCode();

        var dto = await resp.Content.ReadFromJsonAsync<SaseConfigDto>(cancellationToken: ct)
            ?? throw new InvalidOperationException("Empty config response");

        // 3. Edge function tidak return PrivateKey — kita inject dari local KeyStore
        return dto with { PrivateKey = pair.PrivateKey };
    }

    public async Task<bool> IsConfigFreshAsync(CancellationToken ct = default)
    {
        var deviceId = await DeviceIdHelper.GetAsync();
        var req = new HttpRequestMessage(HttpMethod.Get,
            $"{_supabaseUrl}/functions/v1/sase-config/status?device={Uri.EscapeDataString(deviceId)}");
        req.Headers.Add("Authorization", $"Bearer {_userJwtProvider()}");

        using var resp = await _http.SendAsync(req, ct);
        if (!resp.IsSuccessStatusCode) return false;

        var json = await resp.Content.ReadFromJsonAsync<FreshStatus>(cancellationToken: ct);
        return json?.Fresh ?? false;
    }

    private sealed record FreshStatus(bool Fresh, string? Reason);
}
```

`DeviceIdHelper` cara cepat: hash hostname + MAC address pertama:

```csharp
using System.Net.NetworkInformation;
using System.Security.Cryptography;
using System.Text;

namespace HermesNetwork.Sase.Config;

public static class DeviceIdHelper
{
    public static Task<string> GetAsync()
    {
        var hostname = Environment.MachineName;
        var mac = NetworkInterface.GetAllNetworkInterfaces()
            .Where(n => n.NetworkInterfaceType is NetworkInterfaceType.Ethernet or NetworkInterfaceType.Wireless80211)
            .Select(n => n.GetPhysicalAddress().ToString())
            .FirstOrDefault(a => !string.IsNullOrEmpty(a)) ?? "no-mac";

        var raw = $"{hostname}|{mac}";
        var hash = SHA256.HashData(Encoding.UTF8.GetBytes(raw));
        return Task.FromResult(Convert.ToHexString(hash)[..16].ToLowerInvariant());
    }
}
```

## 5.7 Edge Function: `sase-config`

`supabase/functions/sase-config/index.ts`:

```typescript
import { serve } from "https://deno.land/std@0.224.0/http/server.ts";
import { createClient } from "https://esm.sh/@supabase/supabase-js@2.46.1";

const SASE_API_URL          = Deno.env.get("SASE_API_URL")!;
const SASE_API_KEY          = Deno.env.get("SASE_API_KEY")!;
const SASE_GATEWAY_ENDPOINT = Deno.env.get("SASE_GATEWAY_ENDPOINT")!;
const SASE_GATEWAY_PUBKEY   = Deno.env.get("SASE_GATEWAY_PUBLIC_KEY")!;
const SASE_DEFAULT_DNS      = Deno.env.get("SASE_DEFAULT_DNS") || "";

const SUPABASE_URL    = Deno.env.get("SUPABASE_URL")!;
const SERVICE_ROLE    = Deno.env.get("SUPABASE_SERVICE_ROLE_KEY")!;

const ROLE_POLICIES: Record<string, { allowedIps: string; dns?: string; mtu?: number }> = {
  engineer:   { allowedIps: "10.0.0.0/8, 192.168.0.0/16", dns: "10.0.0.1", mtu: 1280 },
  executive:  { allowedIps: "0.0.0.0/0",                  dns: "10.0.0.1", mtu: 1280 },
  intern:     { allowedIps: "10.10.0.0/16",                                  mtu: 1280 },
  contractor: { allowedIps: "10.20.5.0/24",                                  mtu: 1280 },
};

serve(async (req: Request): Promise<Response> => {
  if (req.method === "OPTIONS") return new Response(null, { headers: cors() });
  const url = new URL(req.url);

  // GET /sase-config/status?device=...  -> { fresh, reason }
  if (req.method === "GET" && url.pathname.endsWith("/status")) {
    return await handleStatus(req, url);
  }
  // POST /sase-config  -> SaseConfigDto
  if (req.method === "POST") {
    return await handleRequestConfig(req);
  }
  return jsonError(405, "Method not allowed");
});

async function handleRequestConfig(req: Request): Promise<Response> {
  const userId = await authenticate(req);
  if (!userId) return jsonError(401, "Invalid token");

  const body = await req.json().catch(() => null) as
    { device_id?: string; hostname?: string; public_key?: string; platform?: string } | null;
  if (!body || !body.device_id || !body.public_key) {
    return jsonError(400, "device_id and public_key required");
  }

  const sb = createClient(SUPABASE_URL, SERVICE_ROLE);

  // Lookup user role
  const { data: profile } = await sb.from("user_profiles")
    .select("sase_role")
    .eq("id", userId).single();
  const role = profile?.sase_role ?? "engineer";
  const policy = ROLE_POLICIES[role] ?? ROLE_POLICIES.engineer;

  // Generate PSK & assign IP via SASE control plane
  const peer = await registerPeerAtGateway({
    publicKey: body.public_key,
    role:      role,
    deviceId:  body.device_id,
    userId:    userId,
  });

  // Persist peer record
  await sb.from("sase_peer").upsert({
    user_id:        userId,
    device_id:      body.device_id,
    public_key:     body.public_key,
    psk:            peer.psk,
    interface_ip:   peer.interfaceIp,
    role_at_issue:  role,
    expires_at:     new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString(),
    last_refreshed: new Date().toISOString(),
    status:         "active",
  }, { onConflict: "user_id,device_id" });

  await sb.from("sase_audit_log").insert({
    user_id:   userId,
    device_id: body.device_id,
    action:    "config_request",
    details:   { role, hostname: body.hostname, platform: body.platform },
  });

  // Build response (PrivateKey TIDAK dikirim — client punya sendiri)
  const dto = {
    private_key:       "(client-side)",
    interface_address: `${peer.interfaceIp}/32`,
    dns:               policy.dns ?? SASE_DEFAULT_DNS,
    mtu:               policy.mtu,
    peers: [{
      public_key:        SASE_GATEWAY_PUBKEY,
      preshared_key:     peer.psk,
      endpoint:          SASE_GATEWAY_ENDPOINT,
      allowed_ips:       policy.allowedIps,
      keepalive_seconds: 25,
    }],
    expires_at: new Date(Date.now() + 24 * 60 * 60 * 1000).toISOString(),
  };
  return jsonOk(dto);
}

async function handleStatus(req: Request, url: URL): Promise<Response> {
  const userId = await authenticate(req);
  if (!userId) return jsonError(401, "Invalid token");
  const deviceId = url.searchParams.get("device");
  if (!deviceId) return jsonError(400, "device parameter required");

  const sb = createClient(SUPABASE_URL, SERVICE_ROLE);
  const { data: peer } = await sb.from("sase_peer")
    .select("expires_at, role_at_issue")
    .eq("user_id", userId).eq("device_id", deviceId).single();

  if (!peer) return jsonOk({ fresh: false, reason: "not-registered" });

  // Check role didn't change
  const { data: profile } = await sb.from("user_profiles")
    .select("sase_role").eq("id", userId).single();
  if (profile?.sase_role !== peer.role_at_issue)
    return jsonOk({ fresh: false, reason: "role-changed" });

  const expires = new Date(peer.expires_at).getTime();
  if (Date.now() > expires - 60 * 60 * 1000)   // refresh kalau < 1 jam tersisa
    return jsonOk({ fresh: false, reason: "expiring-soon" });

  return jsonOk({ fresh: true });
}

// === SASE control plane integration (vendor-specific) ===
async function registerPeerAtGateway(input: {
  publicKey: string; role: string; deviceId: string; userId: string;
}): Promise<{ psk: string; interfaceIp: string }> {
  // Contoh — adjust ke API control plane vendor Anda
  const resp = await fetch(`${SASE_API_URL}/peers`, {
    method: "POST",
    headers: {
      "Authorization": `Bearer ${SASE_API_KEY}`,
      "Content-Type":  "application/json",
    },
    body: JSON.stringify({
      public_key: input.publicKey,
      label:      `${input.userId}/${input.deviceId}`,
      tags:       [input.role],
    }),
  });
  if (!resp.ok) {
    const t = await resp.text();
    throw new Error(`SASE register failed: ${resp.status} ${t}`);
  }
  const data = await resp.json();
  return {
    psk:         data.preshared_key,    // base64
    interfaceIp: data.assigned_ip,      // mis. "10.99.0.42"
  };
}

// === helpers ===
async function authenticate(req: Request): Promise<string | null> {
  const auth = req.headers.get("Authorization") ?? "";
  if (!auth.startsWith("Bearer ")) return null;
  const sb = createClient(SUPABASE_URL, SERVICE_ROLE, {
    global: { headers: { Authorization: auth } }
  });
  const { data, error } = await sb.auth.getUser();
  return error ? null : data.user?.id ?? null;
}

function jsonOk(obj: unknown): Response {
  return new Response(JSON.stringify(obj), {
    status: 200,
    headers: { ...cors(), "Content-Type": "application/json" }
  });
}
function jsonError(status: number, message: string): Response {
  return new Response(JSON.stringify({ error: message }), {
    status,
    headers: { ...cors(), "Content-Type": "application/json" }
  });
}
function cors(): HeadersInit {
  return {
    "Access-Control-Allow-Origin":  "*",
    "Access-Control-Allow-Headers": "authorization, x-client-info, apikey, content-type",
    "Access-Control-Allow-Methods": "POST, GET, OPTIONS",
  };
}
```

## 5.8 Deploy edge function

```bash
supabase secrets set SASE_API_URL=https://api.sase-control.example.com
supabase secrets set SASE_API_KEY=your-api-key
supabase secrets set SASE_GATEWAY_ENDPOINT=n1.ndr24.com:51820
supabase secrets set SASE_GATEWAY_PUBLIC_KEY=YourGwPubKeyBase64=
supabase secrets set SASE_DEFAULT_DNS=10.0.0.1,1.1.1.1

supabase functions deploy sase-config
```

Test:

```bash
JWT=$(curl -X POST $SB_URL/auth/v1/token?grant_type=password \
  -H "apikey: $ANON" -H "Content-Type: application/json" \
  -d '{"email":"test@hermes","password":"..."}' | jq -r .access_token)

curl -X POST "$SB_URL/functions/v1/sase-config" \
  -H "Authorization: Bearer $JWT" \
  -H "Content-Type: application/json" \
  -d '{
    "device_id":  "abcd1234",
    "hostname":   "TEST-PC",
    "public_key": "<wg-pubkey-base64>",
    "platform":   "windows"
  }'
```

## 5.9 Best practices

- **Cache config response di memory + apply hanya kalau berubah** — hindari unnecessary tunnel restart
- **Refresh schedule**: cek `IsConfigFreshAsync()` tiap 5 menit, refresh kalau stale
- **Handle 401** dengan re-trigger user login flow
- **Handle 502** (control plane down) dengan exponential back-off, max 3 retry
- **Audit log** semua `config_request` + `config_refresh` di Supabase untuk forensic
- **PSK rotation**: tiap config refresh dapet PSK baru — defense in depth murah

---

[← Bab 4 TunnelSupervisor]({{ site.baseurl }}{% link docs/04-tunnel-supervisor.md %}){: .btn }
[Bab 6 — Connection Flow →]({{ site.baseurl }}{% link docs/06-connection-flow.md %}){: .btn .btn-primary }
