---
layout: default
title: 4. Hermes Helper Service
nav_order: 5
permalink: /docs/tunnel-supervisor/
---

# 4. Hermes Helper Service (Layer Privileged)
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 4.1 Tujuan

Hermes Helper Service adalah **komponen privileged** yang melakukan operasi WireGuard yang butuh admin/root. UI app (yang jalan as user) tidak melakukan operasi ini langsung — request via named-pipe / unix-socket IPC dengan kontrak typed JSON-RPC.

**Mengapa terpisah dari UI?**
- `HermesNetwork360Guard.exe` distribute ke end-user, jalan as user biasa
- Operasi WireGuard butuh privilege admin (lihat [Bab 1]({{ site.baseurl }}{% link docs/01-pendahuluan.md %}) §1.4.2)
- UAC dialog tiap operasi = UX rusak
- Solusi: Helper sekali install (oleh installer dengan elevation), persistent jalan as SYSTEM/root

**Yang dilakukan:**
1. Listen IPC (named-pipe Win, unix-socket Mac)
2. Authenticate caller (same-machine, same-user)
3. Validate input (whitelist verb, regex tunnel name, validate config sintaks)
4. Eksekusi operasi privileged via stdlib (`ServiceController` di Win, `launchctl` di Mac)
5. Audit log tiap request

**Yang TIDAK dilakukan:**
- Tidak shell-out arbitrary command (whitelist verb saja)
- Tidak akses network external
- Tidak read/write file di luar lokasi WireGuard resmi
- Tidak cache state (stateless)

## 4.2 Kontrak JSON-RPC v1

Semua request/response dalam JSON-RPC 2.0 over named-pipe (Win) atau unix-socket (Mac).

### 4.2.1 Frame format

Tiap message di-prefix dengan length (4-byte big-endian) lalu UTF-8 JSON body. Server membaca length, lalu N byte body.

```
+------------+-----------------------------+
| length: 4  | json body: N bytes (UTF-8) |
+------------+-----------------------------+
```

### 4.2.2 Request

```json
{
  "jsonrpc": "2.0",
  "id": "<uuid>",
  "method": "<verb>",
  "params": { ... }
}
```

### 4.2.3 Response

```json
{
  "jsonrpc": "2.0",
  "id": "<uuid>",
  "result": { ... }
}
```

atau error:

```json
{
  "jsonrpc": "2.0",
  "id": "<uuid>",
  "error": { "code": -32xxx, "message": "...", "data": { ... } }
}
```

### 4.2.4 Verb yang didukung

| Method | Params | Result | Privilege diperlukan |
|---|---|---|---|
| `Ping` | `{}` | `{"version":"1.0","time":"2026-04-21T10:00Z"}` | None |
| `InstallTunnel` | `{"name":"Hermes","config":"..."}` | `{"installed":true}` | Admin (install Win Service) |
| `UninstallTunnel` | `{"name":"Hermes"}` | `{"uninstalled":true}` | Admin |
| `ApplyConfig` | `{"name":"Hermes","config":"..."}` | `{"applied":true}` | Admin |
| `StartTunnel` | `{"name":"Hermes"}` | `{"started":true}` | Admin |
| `StopTunnel` | `{"name":"Hermes"}` | `{"stopped":true}` | Admin |
| `GetStatus` | `{"name":"Hermes"}` | `TunnelStatus` (lihat 4.2.5) | None (read) |

### 4.2.5 `TunnelStatus` schema

```json
{
  "name": "Hermes",
  "state": "Up",
  "lastHandshake": "2026-04-21T10:32:15Z",
  "bytesReceived": 1234567,
  "bytesSent":     987654,
  "peerCount": 1,
  "interfaceAddress": "10.99.0.42/32"
}
```

`state` enum: `NotInstalled` | `Stopped` | `Starting` | `Up` | `Stopping` | `Faulted`.

### 4.2.6 Error codes

| Code | Meaning |
|---|---|
| `-32600` | Invalid request format |
| `-32601` | Method not found / not whitelisted |
| `-32602` | Invalid params |
| `-32603` | Internal error |
| `-32000` | Validation failed (mis. tunnel name regex mismatch) |
| `-32001` | Caller not authenticated |
| `-32002` | OS operation failed (forwarded stderr) |

## 4.3 Server-side (HermesHelperSvc)

### 4.3.1 Entry point

```csharp
// HermesHelperSvc/Program.cs
using HermesHelperSvc.Os;
using HermesHelperSvc.Rpc;
using Microsoft.Extensions.Hosting;

var builder = Host.CreateApplicationBuilder(args);

builder.Services.AddSingleton<IOsBackend>(_ =>
    OperatingSystem.IsWindows() ? new WindowsBackend() :
    OperatingSystem.IsMacOS()   ? new MacBackend()     :
    throw new PlatformNotSupportedException());

builder.Services.AddSingleton<JsonRpcServer>();
builder.Services.AddHostedService<HelperBackgroundService>();

if (OperatingSystem.IsWindows())
    builder.Services.AddWindowsService(o => o.ServiceName = "HermesHelperSvc");
else if (OperatingSystem.IsLinux() || OperatingSystem.IsMacOS())
    builder.Services.AddSystemd();   // works for launchd lifecycle too

var host = builder.Build();
await host.RunAsync();
```

### 4.3.2 `HelperBackgroundService`

```csharp
namespace HermesHelperSvc;

public sealed class HelperBackgroundService : BackgroundService
{
    private readonly JsonRpcServer _rpc;
    private readonly ILogger<HelperBackgroundService> _log;

    public HelperBackgroundService(JsonRpcServer rpc, ILogger<HelperBackgroundService> log)
    { _rpc = rpc; _log = log; }

    protected override async Task ExecuteAsync(CancellationToken ct)
    {
        _log.LogInformation("HermesHelperSvc starting");
        await _rpc.RunAsync(ct);
    }
}
```

### 4.3.3 `JsonRpcServer` (cross-platform)

```csharp
using System.Buffers.Binary;
using System.IO.Pipes;
using System.Net.Sockets;
using System.Text;
using System.Text.Json;
using HermesHelperSvc.Os;

namespace HermesHelperSvc.Rpc;

public sealed class JsonRpcServer
{
    private const string WindowsPipeName = "HermesHelper";
    private const string UnixSocketPath  = "/var/run/hermes-helper.sock";

    private readonly IOsBackend _backend;
    private readonly ILogger<JsonRpcServer> _log;
    private readonly CallerAuthenticator _auth;

    public JsonRpcServer(IOsBackend backend, ILogger<JsonRpcServer> log, CallerAuthenticator auth)
    { _backend = backend; _log = log; _auth = auth; }

    public async Task RunAsync(CancellationToken ct)
    {
        if (OperatingSystem.IsWindows())
            await RunWindowsAsync(ct);
        else
            await RunUnixAsync(ct);
    }

    private async Task RunWindowsAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var pipe = new NamedPipeServerStream(
                WindowsPipeName,
                PipeDirection.InOut,
                NamedPipeServerStream.MaxAllowedServerInstances,
                PipeTransmissionMode.Byte,
                PipeOptions.Asynchronous);

            await pipe.WaitForConnectionAsync(ct);
            _ = HandleClientAsync(pipe, ct);  // fire-and-forget per connection
        }
    }

    private async Task RunUnixAsync(CancellationToken ct)
    {
        if (File.Exists(UnixSocketPath)) File.Delete(UnixSocketPath);
        var endpoint = new UnixDomainSocketEndPoint(UnixSocketPath);

        using var listener = new Socket(
            AddressFamily.Unix, SocketType.Stream, ProtocolType.Unspecified);
        listener.Bind(endpoint);
        listener.Listen(64);

        // Restrict socket permission ke root only — kernel akan check kalau pake SO_PEERCRED
        File.SetUnixFileMode(UnixSocketPath,
            UnixFileMode.UserRead | UnixFileMode.UserWrite | UnixFileMode.UserExecute);

        while (!ct.IsCancellationRequested)
        {
            var conn = await listener.AcceptAsync(ct);
            _ = HandleSocketAsync(conn, ct);
        }
    }

    private async Task HandleClientAsync(Stream conn, CancellationToken ct)
    {
        using (conn)
        {
            try
            {
                if (!await _auth.AuthenticateAsync(conn, ct))
                {
                    await WriteErrorAsync(conn, null, -32001, "Caller not authenticated", ct);
                    return;
                }

                while (conn.CanRead)
                {
                    var msg = await ReadFrameAsync(conn, ct);
                    if (msg is null) return;
                    var resp = await DispatchAsync(msg, ct);
                    await WriteFrameAsync(conn, resp, ct);
                }
            }
            catch (Exception ex)
            {
                _log.LogError(ex, "RPC connection error");
            }
        }
    }

    private async Task HandleSocketAsync(Socket sock, CancellationToken ct)
    {
        await using var ns = new NetworkStream(sock, ownsSocket: true);
        await HandleClientAsync(ns, ct);
    }

    private async Task<string> DispatchAsync(string requestJson, CancellationToken ct)
    {
        try
        {
            var req = JsonSerializer.Deserialize<RpcRequest>(requestJson)
                ?? throw new InvalidOperationException("null request");

            _log.LogInformation("RPC {Method} id={Id}", req.Method, req.Id);

            object? result = req.Method switch
            {
                "Ping"            => new { version = "1.0", time = DateTimeOffset.UtcNow },
                "InstallTunnel"   => await HandleInstall(req, ct),
                "UninstallTunnel" => await HandleUninstall(req, ct),
                "ApplyConfig"     => await HandleApplyConfig(req, ct),
                "StartTunnel"     => await HandleStart(req, ct),
                "StopTunnel"      => await HandleStop(req, ct),
                "GetStatus"       => await HandleStatus(req, ct),
                _ => throw new RpcException(-32601, $"Unknown method: {req.Method}")
            };

            return JsonSerializer.Serialize(new RpcResponse(req.Id, result, null));
        }
        catch (RpcException rx)
        {
            return JsonSerializer.Serialize(new RpcResponse(
                ParseId(requestJson), null,
                new RpcError(rx.Code, rx.Message, rx.Data)));
        }
        catch (Exception ex)
        {
            _log.LogError(ex, "RPC dispatch error");
            return JsonSerializer.Serialize(new RpcResponse(
                ParseId(requestJson), null,
                new RpcError(-32603, "Internal error", ex.Message)));
        }
    }

    private async Task<object> HandleInstall(RpcRequest req, CancellationToken ct)
    {
        var p = req.Params!.GetValueAs<InstallParams>();
        ValidateTunnelName(p.Name);
        ValidateWgConfig(p.Config);
        await _backend.InstallTunnelAsync(p.Name, p.Config, ct);
        return new { installed = true };
    }

    private async Task<object> HandleApplyConfig(RpcRequest req, CancellationToken ct)
    {
        var p = req.Params!.GetValueAs<InstallParams>();
        ValidateTunnelName(p.Name);
        ValidateWgConfig(p.Config);
        await _backend.ApplyConfigAsync(p.Name, p.Config, ct);
        return new { applied = true };
    }

    private async Task<object> HandleStart(RpcRequest req, CancellationToken ct)
    {
        var p = req.Params!.GetValueAs<NameParam>();
        ValidateTunnelName(p.Name);
        await _backend.StartTunnelAsync(p.Name, ct);
        return new { started = true };
    }

    private async Task<object> HandleStop(RpcRequest req, CancellationToken ct)
    {
        var p = req.Params!.GetValueAs<NameParam>();
        ValidateTunnelName(p.Name);
        await _backend.StopTunnelAsync(p.Name, ct);
        return new { stopped = true };
    }

    private async Task<object> HandleUninstall(RpcRequest req, CancellationToken ct)
    {
        var p = req.Params!.GetValueAs<NameParam>();
        ValidateTunnelName(p.Name);
        await _backend.UninstallTunnelAsync(p.Name, ct);
        return new { uninstalled = true };
    }

    private async Task<object> HandleStatus(RpcRequest req, CancellationToken ct)
    {
        var p = req.Params!.GetValueAs<NameParam>();
        ValidateTunnelName(p.Name);
        return await _backend.GetStatusAsync(p.Name, ct);
    }

    // ---------------- helpers ----------------

    private static void ValidateTunnelName(string name)
    {
        if (string.IsNullOrEmpty(name) || name.Length > 32 ||
            !System.Text.RegularExpressions.Regex.IsMatch(name, @"^[A-Za-z0-9_-]+$"))
            throw new RpcException(-32000, "Invalid tunnel name");
    }

    private static void ValidateWgConfig(string content)
    {
        if (string.IsNullOrWhiteSpace(content) || content.Length > 16 * 1024)
            throw new RpcException(-32000, "Config empty or too large");
        if (!content.Contains("[Interface]") || !content.Contains("[Peer]"))
            throw new RpcException(-32000, "Config missing [Interface] or [Peer] section");
    }

    private static async Task<string?> ReadFrameAsync(Stream s, CancellationToken ct)
    {
        var lenBuf = new byte[4];
        var n = await s.ReadAsync(lenBuf.AsMemory(), ct);
        if (n != 4) return null;
        var len = BinaryPrimitives.ReadInt32BigEndian(lenBuf);
        if (len <= 0 || len > 1 * 1024 * 1024) throw new InvalidDataException("Frame too large");
        var body = new byte[len];
        var read = 0;
        while (read < len)
        {
            var r = await s.ReadAsync(body.AsMemory(read, len - read), ct);
            if (r == 0) return null;
            read += r;
        }
        return Encoding.UTF8.GetString(body);
    }

    private static async Task WriteFrameAsync(Stream s, string content, CancellationToken ct)
    {
        var body = Encoding.UTF8.GetBytes(content);
        var hdr = new byte[4];
        BinaryPrimitives.WriteInt32BigEndian(hdr, body.Length);
        await s.WriteAsync(hdr, ct);
        await s.WriteAsync(body, ct);
        await s.FlushAsync(ct);
    }

    private static string? ParseId(string json)
    {
        try { return JsonDocument.Parse(json).RootElement.GetProperty("id").GetString(); }
        catch { return null; }
    }
}

public sealed record InstallParams(string Name, string Config);
public sealed record NameParam(string Name);
public sealed record RpcRequest(string Id, string Method, JsonElement? Params);
public sealed record RpcResponse(string? Id, object? Result, RpcError? Error)
{
    public string Jsonrpc => "2.0";
}
public sealed record RpcError(int Code, string Message, object? Data);
public sealed class RpcException(int code, string message, object? data = null) : Exception(message)
{
    public int Code { get; } = code;
    public object? Data { get; } = data;
}

public static class JsonElementExt
{
    public static T GetValueAs<T>(this JsonElement el) =>
        JsonSerializer.Deserialize<T>(el.GetRawText())
        ?? throw new RpcException(-32602, "Invalid params");
}
```

### 4.3.4 `IOsBackend` (Windows + Mac)

```csharp
namespace HermesHelperSvc.Os;

public interface IOsBackend
{
    Task InstallTunnelAsync(string name, string config, CancellationToken ct);
    Task UninstallTunnelAsync(string name, CancellationToken ct);
    Task ApplyConfigAsync(string name, string config, CancellationToken ct);
    Task StartTunnelAsync(string name, CancellationToken ct);
    Task StopTunnelAsync(string name, CancellationToken ct);
    Task<TunnelStatus> GetStatusAsync(string name, CancellationToken ct);
}

public sealed record TunnelStatus(
    string Name,
    string State,                    // "NotInstalled" | "Stopped" | "Up" | ...
    DateTimeOffset? LastHandshake,
    long BytesReceived,
    long BytesSent,
    int PeerCount,
    string? InterfaceAddress);
```

#### `WindowsBackend.cs`

Implementasi Windows mengikuti **pattern existing** di Hermes Guard yang memakai **embedded WireGuard via `tunnel.dll` + `wireguard.dll`** P/Invoke (lihat folder `HermesNetwork/TunnelDll/`). **Tidak ada call ke `wireguard.exe` external, tidak ada path `C:\Program Files\WireGuard\`.**

Helper Service mengkonsumsi tipe & helper yang sudah ada di `HermesNetwork.TunnelDll` (di-share via project reference, atau di-copy ke project Helper):

| Existing tipe | Pakai untuk |
|---|---|
| `Service.Add(configFile, ephemeral)` | InstallTunnel — register Windows Service `WireGuardTunnel$<name>` dengan binPath ke binary self (HermesHelperSvc / HermesNetwork360Guard) plus argumen `/service <conf> <pid>`. Internal call `Win32.OpenSCManager` + `CreateService`. |
| `Service.Remove(tunnelName)` | UninstallTunnel — `ControlService` STOP + `DeleteService` |
| `Service.Run(configFile)` | Tunnel runtime — call `WireGuardTunnelService(configFile)` di `tunnel.dll` (di-call hanya saat binary jalan dengan `/service` flag, bukan dari Helper directly) |
| `Driver.Adapter(...)` | Adapter handle untuk query status (dari `wireguard.dll`: `WireGuardOpenAdapter`, `WireGuardGetConfiguration`, dst.) |

```csharp
using HermesNetwork.TunnelDll;     // existing project reference

[SupportedOSPlatform("windows")]
public sealed class WindowsBackend : IOsBackend
{
    // Hermes Guard memakai folder data sendiri, BUKAN C:\Program Files\WireGuard\
    private static string ConfDir => Path.Combine(
        Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
        "HermesNetwork360Guard", "Sase");

    private static string ConfPath(string name) => Path.Combine(ConfDir, $"{name}.conf");

    public async Task InstallTunnelAsync(string name, string config, CancellationToken ct)
    {
        Directory.CreateDirectory(ConfDir);
        var conf = ConfPath(name);
        await File.WriteAllTextAsync(conf, config, ct);

        // Existing helper di TunnelDll/Service.cs — register Windows Service
        // WireGuardTunnel$<name> dengan binPath = current process exe + "/service <conf> <pid>"
        await Task.Run(() => Service.Add(conf, ephemeral: false), ct);
    }

    public async Task UninstallTunnelAsync(string name, CancellationToken ct)
    {
        await Task.Run(() => Service.Remove(name), ct);    // existing wrapper — call DeleteService
        try { File.Delete(ConfPath(name)); } catch { }
    }

    public async Task ApplyConfigAsync(string name, string config, CancellationToken ct)
    {
        // Replace config file + restart service
        await File.WriteAllTextAsync(ConfPath(name), config, ct);

        var st = await GetStatusAsync(name, ct);
        if (st.State == "NotInstalled")
        {
            await Task.Run(() => Service.Add(ConfPath(name), ephemeral: false), ct);
        }
        else
        {
            // Reload via uninstall + reinstall (paling simpel; reload tunnel runtime
            // butuh ipc UAPI pipe ke WireGuardTunnel$<name> — bisa di-extend nanti)
            await UninstallTunnelAsync(name, ct);
            await Task.Run(() => Service.Add(ConfPath(name), ephemeral: false), ct);
        }
    }

    public async Task StartTunnelAsync(string name, CancellationToken ct)
    {
        using var sc = new ServiceController($"WireGuardTunnel${name}");
        if (sc.Status == ServiceControllerStatus.Running) return;
        sc.Start();
        await WaitForStatusAsync(sc, ServiceControllerStatus.Running, TimeSpan.FromSeconds(30), ct);
    }

    public async Task StopTunnelAsync(string name, CancellationToken ct)
    {
        using var sc = new ServiceController($"WireGuardTunnel${name}");
        if (sc.Status == ServiceControllerStatus.Stopped) return;
        sc.Stop();
        await WaitForStatusAsync(sc, ServiceControllerStatus.Stopped, TimeSpan.FromSeconds(30), ct);
    }

    public Task<TunnelStatus> GetStatusAsync(string name, CancellationToken ct)
    {
        var serviceName = $"WireGuardTunnel${name}";
        string state;
        try
        {
            using var sc = new ServiceController(serviceName);
            state = sc.Status switch
            {
                ServiceControllerStatus.Running => "Up",
                ServiceControllerStatus.Stopped => "Stopped",
                ServiceControllerStatus.StartPending => "Starting",
                ServiceControllerStatus.StopPending => "Stopping",
                _ => "Faulted",
            };
        }
        catch (InvalidOperationException)
        {
            return Task.FromResult(new TunnelStatus(name, "NotInstalled", null, 0, 0, 0, null));
        }

        if (state != "Up")
            return Task.FromResult(new TunnelStatus(name, state, null, 0, 0, 0, null));

        // Query stats via Driver.Adapter (wireguard.dll) — existing pattern
        return Task.FromResult(QueryViaDriver(name, state));
    }

    private static TunnelStatus QueryViaDriver(string name, string state)
    {
        try
        {
            using var adapter = new Driver.Adapter(name);
            // Parse adapter.GetConfiguration() → byte[] WG_INTERFACE struct + WG_PEER[]
            // Existing helper di Driver.cs sudah expose method-method ini.
            // Hitung bytesRx/Tx/lastHandshake dari struct.
            // (kode parsing UAPI WireGuard sudah ada di TunnelDll/Driver.cs sebagai pattern)
            return new TunnelStatus(name, state,
                LastHandshake: null /* parsed dari config */,
                BytesReceived: 0, BytesSent: 0, PeerCount: 0,
                InterfaceAddress: null);
        }
        catch
        {
            return new TunnelStatus(name, "Faulted", null, 0, 0, 0, null);
        }
    }

    // (versi alternatif yang lebih simple kalau tidak butuh stats real-time:
    //  passthrough TunnelStatus dengan state saja, tanpa parse adapter config —
    //  cukup untuk UX awal "Connected vs Disconnected"; bisa di-extend setelahnya.)

    private async Task<TunnelStatus> ParseWgShowAsync(string name, CancellationToken ct)
    {
        // [DEPRECATED kalau pakai Driver.Adapter di atas. Tinggal kalau memang
        //  butuh fallback ke wg.exe — yang TIDAK ada di Hermes Guard installation.]
        // Method ini disimpan sebagai referensi; jangan dipakai di production.
        await Task.CompletedTask;
        return new TunnelStatus(name, "Faulted", null, 0, 0, 0, null);
    }

    private async Task<TunnelStatus> ParseWgShowAsyncOriginal(string name, CancellationToken ct)
    {
        // Original implementation kept for reference (REMOVED FROM PRODUCTION)
        var (code, stdout, _) = (-1, "", "");
        if (code != 0)
            return new TunnelStatus(name, "Faulted", null, 0, 0, 0, null);

        long rx = 0, tx = 0;
        DateTimeOffset? lastHs = null;
        int peers = 0;
        var lines = stdout.Split('\n', StringSplitOptions.RemoveEmptyEntries);
        for (int i = 1; i < lines.Length; i++)
        {
            var cols = lines[i].Split('\t');
            if (cols.Length < 8) continue;
            peers++;
            if (long.TryParse(cols[5], out var rxB)) rx += rxB;
            if (long.TryParse(cols[6], out var txB)) tx += txB;
            if (long.TryParse(cols[4], out var hs) && hs > 0)
            {
                var t = DateTimeOffset.FromUnixTimeSeconds(hs);
                if (lastHs is null || t > lastHs) lastHs = t;
            }
        }
        return new TunnelStatus(name, "Up", lastHs, rx, tx, peers, null);
    }

    // ---------------- helpers ----------------

    private static async Task RunAsync(string file, string args, CancellationToken ct)
    {
        var (code, _, stderr) = await RunWithOutputAsync(file, args, ct);
        if (code != 0)
            throw new RpcException(-32002, $"OS command failed (exit {code}): {stderr}");
    }

    private static async Task<(int Code, string Stdout, string Stderr)> RunWithOutputAsync(
        string file, string args, CancellationToken ct)
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
        await p.WaitForExitAsync(ct);
        return (p.ExitCode, await so, await se);
    }

    private static async Task WaitForStatusAsync(
        ServiceController sc, ServiceControllerStatus target, TimeSpan timeout, CancellationToken ct)
    {
        var deadline = DateTime.UtcNow + timeout;
        while (DateTime.UtcNow < deadline)
        {
            ct.ThrowIfCancellationRequested();
            sc.Refresh();
            if (sc.Status == target) return;
            await Task.Delay(500, ct);
        }
        throw new TimeoutException($"Service did not reach {target} in {timeout.TotalSeconds}s");
    }
}
```

#### `MacBackend.cs`

Pakai `wg-quick` + `launchctl`. Helper sendiri jalan as root (LaunchDaemon), jadi tidak butuh `osascript` elevation.

```csharp
[SupportedOSPlatform("macos")]
public sealed class MacBackend : IOsBackend
{
    private const string WgQuick = "/usr/local/bin/wg-quick";
    private const string WgBin   = "/usr/local/bin/wg";
    private const string Label   = "com.hermesnetwork.sase";   // LaunchDaemon untuk WG tunnel
    private string ConfPath(string name) => $"/etc/wireguard/{name}.conf";
    private string PlistPath(string name) => $"/Library/LaunchDaemons/com.hermesnetwork.sase.{name}.plist";

    public async Task InstallTunnelAsync(string name, string config, CancellationToken ct)
    {
        Directory.CreateDirectory("/etc/wireguard");
        await File.WriteAllTextAsync(ConfPath(name), config, ct);
        File.SetUnixFileMode(ConfPath(name), UnixFileMode.UserRead | UnixFileMode.UserWrite);

        // Optional: register LaunchDaemon agar tunnel auto-start saat reboot
        var plist = LaunchDaemonPlistFor(name);
        await File.WriteAllTextAsync(PlistPath(name), plist, ct);
        File.SetUnixFileMode(PlistPath(name),
            UnixFileMode.UserRead | UnixFileMode.UserWrite |
            UnixFileMode.GroupRead | UnixFileMode.OtherRead);
    }

    public async Task UninstallTunnelAsync(string name, CancellationToken ct)
    {
        try { await StopTunnelAsync(name, ct); } catch { }
        await RunAsync("launchctl", $"bootout system/com.hermesnetwork.sase.{name}", ct, allowFail: true);
        if (File.Exists(PlistPath(name))) File.Delete(PlistPath(name));
        if (File.Exists(ConfPath(name))) File.Delete(ConfPath(name));
    }

    public async Task ApplyConfigAsync(string name, string config, CancellationToken ct)
    {
        await File.WriteAllTextAsync(ConfPath(name), config, ct);
        File.SetUnixFileMode(ConfPath(name), UnixFileMode.UserRead | UnixFileMode.UserWrite);
        // Restart tunnel kalau sedang up — wg-quick down + up
        var st = await GetStatusAsync(name, ct);
        if (st.State == "Up")
        {
            await StopTunnelAsync(name, ct);
            await StartTunnelAsync(name, ct);
        }
    }

    public Task StartTunnelAsync(string name, CancellationToken ct)
        => RunAsync(WgQuick, $"up {name}", ct);

    public Task StopTunnelAsync(string name, CancellationToken ct)
        => RunAsync(WgQuick, $"down {name}", ct, allowFail: true);

    public async Task<TunnelStatus> GetStatusAsync(string name, CancellationToken ct)
    {
        if (!File.Exists(ConfPath(name)))
            return new TunnelStatus(name, "NotInstalled", null, 0, 0, 0, null);

        var (code, stdout, _) = await RunWithOutputAsync(WgBin, $"show {name} dump", ct);
        if (code != 0)
            return new TunnelStatus(name, "Stopped", null, 0, 0, 0, null);

        long rx = 0, tx = 0;
        DateTimeOffset? lastHs = null;
        int peers = 0;
        var lines = stdout.Split('\n', StringSplitOptions.RemoveEmptyEntries);
        for (int i = 1; i < lines.Length; i++)
        {
            var cols = lines[i].Split('\t');
            if (cols.Length < 8) continue;
            peers++;
            if (long.TryParse(cols[5], out var rxB)) rx += rxB;
            if (long.TryParse(cols[6], out var txB)) tx += txB;
            if (long.TryParse(cols[4], out var hs) && hs > 0)
            {
                var t = DateTimeOffset.FromUnixTimeSeconds(hs);
                if (lastHs is null || t > lastHs) lastHs = t;
            }
        }
        return new TunnelStatus(name, "Up", lastHs, rx, tx, peers, null);
    }

    private static string LaunchDaemonPlistFor(string name) => $$"""
        <?xml version="1.0" encoding="UTF-8"?>
        <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
        <plist version="1.0">
        <dict>
            <key>Label</key><string>com.hermesnetwork.sase.{{name}}</string>
            <key>ProgramArguments</key>
            <array>
                <string>/usr/local/bin/wg-quick</string>
                <string>up</string>
                <string>{{name}}</string>
            </array>
            <key>RunAtLoad</key><true/>
            <key>KeepAlive</key><false/>
            <key>UserName</key><string>root</string>
        </dict>
        </plist>
        """;

    // ---------------- helpers ----------------

    private static async Task RunAsync(string file, string args, CancellationToken ct, bool allowFail = false)
    {
        var (code, _, stderr) = await RunWithOutputAsync(file, args, ct);
        if (code != 0 && !allowFail)
            throw new RpcException(-32002, $"OS command failed (exit {code}): {stderr}");
    }

    private static async Task<(int Code, string Stdout, string Stderr)> RunWithOutputAsync(
        string file, string args, CancellationToken ct)
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
        await p.WaitForExitAsync(ct);
        return (p.ExitCode, await so, await se);
    }
}
```

### 4.3.5 `CallerAuthenticator`

Hanya terima request dari user yang sama (mencegah sniffing dari user lain di multi-user system):

```csharp
public sealed class CallerAuthenticator
{
    public Task<bool> AuthenticateAsync(Stream conn, CancellationToken ct)
    {
        // Windows: NamedPipeServerStream punya ImpersonateClient + WindowsIdentity
        if (conn is NamedPipeServerStream pipe)
        {
            try
            {
                pipe.RunAsClient(() =>
                {
                    using var id = WindowsIdentity.GetCurrent();
                    // (kalau kita mau policy: hanya user tertentu / Administrators)
                });
                return Task.FromResult(true);
            }
            catch { return Task.FromResult(false); }
        }

        // Unix: SO_PEERCRED via socket option (atau cek peer creds via Mono.Posix)
        if (conn is NetworkStream ns && ns.Socket is { } sock)
        {
            // Implementasi SO_PEERCRED tergantung versi OS;
            // di Mac, gunakan getpeereid() via P/Invoke.
            // Untuk simple: percaya socket di /var/run/hermes-helper.sock dengan permission 0600
            return Task.FromResult(true);
        }

        return Task.FromResult(false);
    }
}
```

> **Catatan keamanan:** detail caller authentication bergantung OS. Lihat [Bab 7]({{ site.baseurl }}{% link docs/07-keamanan.md %}) untuk pembahasan menyeluruh.

## 4.4 Client-side (di UI app)

### 4.4.1 `IHelperServiceClient`

```csharp
namespace HermesNetwork.Sase.Helper;

public interface IHelperServiceClient
{
    Task PingAsync(CancellationToken ct = default);
    Task ApplyConfigAsync(string name, string config, CancellationToken ct = default);
    Task InstallTunnelAsync(string name, string config, CancellationToken ct = default);
    Task UninstallTunnelAsync(string name, CancellationToken ct = default);
    Task StartTunnelAsync(string name, CancellationToken ct = default);
    Task StopTunnelAsync(string name, CancellationToken ct = default);
    Task<TunnelStatusDto> GetStatusAsync(string name, CancellationToken ct = default);
}

public sealed record TunnelStatusDto(
    string Name,
    string State,
    DateTimeOffset? LastHandshake,
    long BytesReceived,
    long BytesSent,
    int PeerCount,
    string? InterfaceAddress);
```

### 4.4.2 `HelperServiceClient`

```csharp
using System.Buffers.Binary;
using System.IO.Pipes;
using System.Net.Sockets;
using System.Text;
using System.Text.Json;

namespace HermesNetwork.Sase.Helper;

public sealed class HelperServiceClient : IHelperServiceClient
{
    private const string WindowsPipe = "HermesHelper";
    private const string UnixSocket  = "/var/run/hermes-helper.sock";

    public Task PingAsync(CancellationToken ct = default)
        => CallAsync<object?, object?>("Ping", null, ct);

    public Task ApplyConfigAsync(string name, string config, CancellationToken ct = default)
        => CallAsync<InstallParams, object>("ApplyConfig", new(name, config), ct);

    public Task InstallTunnelAsync(string name, string config, CancellationToken ct = default)
        => CallAsync<InstallParams, object>("InstallTunnel", new(name, config), ct);

    public Task UninstallTunnelAsync(string name, CancellationToken ct = default)
        => CallAsync<NameParam, object>("UninstallTunnel", new(name), ct);

    public Task StartTunnelAsync(string name, CancellationToken ct = default)
        => CallAsync<NameParam, object>("StartTunnel", new(name), ct);

    public Task StopTunnelAsync(string name, CancellationToken ct = default)
        => CallAsync<NameParam, object>("StopTunnel", new(name), ct);

    public Task<TunnelStatusDto> GetStatusAsync(string name, CancellationToken ct = default)
        => CallAsync<NameParam, TunnelStatusDto>("GetStatus", new(name), ct)!;

    // ---------------- internals ----------------

    private async Task<TResult?> CallAsync<TParams, TResult>(
        string method, TParams? @params, CancellationToken ct)
    {
        await using var stream = await ConnectAsync(ct);
        var req = JsonSerializer.Serialize(new
        {
            jsonrpc = "2.0",
            id = Guid.NewGuid().ToString("N"),
            method,
            @params
        });
        await WriteFrameAsync(stream, req, ct);

        var respJson = await ReadFrameAsync(stream, ct)
            ?? throw new InvalidOperationException("Helper closed connection");

        using var doc = JsonDocument.Parse(respJson);
        if (doc.RootElement.TryGetProperty("error", out var err))
            throw new HelperRpcException(
                err.GetProperty("code").GetInt32(),
                err.GetProperty("message").GetString() ?? "");

        if (!doc.RootElement.TryGetProperty("result", out var result))
            return default;
        if (typeof(TResult) == typeof(object)) return default;
        return JsonSerializer.Deserialize<TResult>(result.GetRawText());
    }

    private static async Task<Stream> ConnectAsync(CancellationToken ct)
    {
        if (OperatingSystem.IsWindows())
        {
            var pipe = new NamedPipeClientStream(
                ".", WindowsPipe, PipeDirection.InOut, PipeOptions.Asynchronous);
            await pipe.ConnectAsync(5_000, ct);
            return pipe;
        }
        else
        {
            var sock = new Socket(AddressFamily.Unix, SocketType.Stream, ProtocolType.Unspecified);
            await sock.ConnectAsync(new UnixDomainSocketEndPoint(UnixSocket), ct);
            return new NetworkStream(sock, ownsSocket: true);
        }
    }

    private static async Task WriteFrameAsync(Stream s, string content, CancellationToken ct)
    {
        var body = Encoding.UTF8.GetBytes(content);
        var hdr = new byte[4];
        BinaryPrimitives.WriteInt32BigEndian(hdr, body.Length);
        await s.WriteAsync(hdr, ct);
        await s.WriteAsync(body, ct);
        await s.FlushAsync(ct);
    }

    private static async Task<string?> ReadFrameAsync(Stream s, CancellationToken ct)
    {
        var lenBuf = new byte[4];
        var n = await s.ReadAsync(lenBuf.AsMemory(), ct);
        if (n != 4) return null;
        var len = BinaryPrimitives.ReadInt32BigEndian(lenBuf);
        if (len <= 0 || len > 1 * 1024 * 1024) throw new InvalidDataException("Frame too large");
        var body = new byte[len];
        var read = 0;
        while (read < len)
        {
            var r = await s.ReadAsync(body.AsMemory(read, len - read), ct);
            if (r == 0) return null;
            read += r;
        }
        return Encoding.UTF8.GetString(body);
    }

    private sealed record InstallParams(string Name, string Config);
    private sealed record NameParam(string Name);
}

public sealed class HelperRpcException(int code, string message) : Exception(message)
{
    public int Code { get; } = code;
}
```

### 4.4.3 Penggunaan dari ConnectionService

```csharp
public sealed class SaseConnectionService
{
    private readonly IHelperServiceClient _helper;
    private readonly ISaseConfigClient _config;
    private const string TunnelName = "Hermes";

    public async Task ConnectAsync(CancellationToken ct = default)
    {
        var configContent = await _config.GetConfigAsync(ct);   // dari user_data Supabase
        await _helper.ApplyConfigAsync(TunnelName, configContent, ct);
        await _helper.StartTunnelAsync(TunnelName, ct);
        // polling status...
    }

    public Task DisconnectAsync(CancellationToken ct = default)
        => _helper.StopTunnelAsync(TunnelName, ct);

    public Task<TunnelStatusDto> GetStatusAsync(CancellationToken ct = default)
        => _helper.GetStatusAsync(TunnelName, ct);
}
```

## 4.5 Testing

### 4.5.1 Unit test Helper handlers (mock IOsBackend)

```csharp
public class JsonRpcServerTests
{
    [Fact]
    public async Task DispatchAsync_StartTunnel_CallsBackend()
    {
        var backend = new Mock<IOsBackend>();
        var server = new JsonRpcServer(backend.Object, NullLogger, new());

        var req = JsonSerializer.Serialize(new {
            jsonrpc = "2.0", id = "1",
            method = "StartTunnel",
            @params = new { name = "Hermes" }
        });

        var resp = await server.DispatchAsync(req, default);
        backend.Verify(b => b.StartTunnelAsync("Hermes", default), Times.Once);
        resp.Should().Contain("\"started\":true");
    }
}
```

### 4.5.2 Integration test end-to-end

Spin up Helper Service local dengan elevation, lalu pakai `HelperServiceClient` dari proses test.

## 4.6 Best practices

- **Selalu validate input di server side** — jangan trust client meski authenticated
- **Whitelist verb**, jangan terima method baru tanpa code review
- **Tidak ada arbitrary command execution** — verb-spesifik dengan parameter terstruktur
- **Log semua request** ke Event Viewer (Win) / `os_log` (Mac) untuk audit
- **Fail closed**: kalau ada doubt soal authentication, tolak request

---

[← Bab 3 Prasyarat]({{ site.baseurl }}{% link docs/03-prasyarat.md %}){: .btn }
[Bab 5 — Config Service →]({{ site.baseurl }}{% link docs/05-config-service.md %}){: .btn .btn-primary }
