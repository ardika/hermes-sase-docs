---
layout: default
title: 4. Layer 1 — TunnelSupervisor
nav_order: 5
permalink: /docs/tunnel-supervisor/
---

# 4. Layer 1 — TunnelSupervisor
{: .no_toc }

## Daftar Isi
{: .no_toc .text-delta }

1. TOC
{:toc}

---

## 4.1 Tujuan

`ITunnelSupervisor` adalah abstraksi cross-platform untuk **lifecycle WireGuard tunnel di OS lokal**. Tugasnya:

1. Apply / Replace config tunnel
2. Start / Stop tunnel (dan service yang membungkusnya)
3. Query status: up/down, last handshake, bytes transferred, peer count
4. Install / Uninstall tunnel service

**Yang TIDAK menjadi tugas layer ini:**
- Generate config (itu Layer 2 — `SaseConfigClient`)
- Tahu detail backend SASE
- Manage user identity / authentication
- Logging server-side

## 4.2 Interface

```csharp
using System;
using System.Threading;
using System.Threading.Tasks;

namespace HermesNetwork.Sase.Supervisor;

public interface ITunnelSupervisor
{
    /// <summary>Nama logikal tunnel (mis. "Hermes"). Sama dengan label OS service.</summary>
    string TunnelName { get; }

    /// <summary>Apply / replace config tunnel. Tidak start tunnel.</summary>
    Task ApplyConfigAsync(string wgConfig, CancellationToken ct = default);

    /// <summary>Start tunnel. Idempotent.</summary>
    Task StartAsync(CancellationToken ct = default);

    /// <summary>Stop tunnel. Idempotent.</summary>
    Task StopAsync(CancellationToken ct = default);

    /// <summary>Uninstall tunnel service + delete config file.</summary>
    Task UninstallAsync(CancellationToken ct = default);

    /// <summary>Query status sekarang. Cepat (sub-second).</summary>
    Task<TunnelStatus> GetStatusAsync(CancellationToken ct = default);

    /// <summary>Tunggu sampai handshake pertama / berikutnya berhasil (timeout = throw).</summary>
    Task<DateTimeOffset> WaitHandshakeAsync(
        TimeSpan? timeout = null, CancellationToken ct = default);

    /// <summary>True kalau process berjalan dengan privilege admin/root.</summary>
    bool IsElevated { get; }
}

public sealed record TunnelStatus(
    TunnelState State,
    DateTimeOffset? LastHandshake,
    long BytesReceived,
    long BytesSent,
    int PeerCount,
    string? LocalIp,
    string? PublicIp);

public enum TunnelState
{
    NotInstalled,
    Stopped,
    Starting,
    Up,
    Stopping,
    Faulted          // service crashed / config invalid
}
```

## 4.3 Pemilihan implementasi runtime

```csharp
namespace HermesNetwork.Sase.Supervisor;

public static class TunnelSupervisorFactory
{
    public static ITunnelSupervisor Create(string tunnelName)
    {
        if (OperatingSystem.IsWindows())
            return new Windows.WindowsTunnelSupervisor(tunnelName);
        if (OperatingSystem.IsMacOS())
            return new Mac.MacTunnelSupervisor(tunnelName);

        throw new PlatformNotSupportedException(
            "TunnelSupervisor only supports Windows and macOS.");
    }
}
```

## 4.4 Implementasi Windows

WireGuard for Windows menyediakan **named-pipe protocol** dan command line `wireguard.exe /installtunnelservice <conf>` / `/uninstalltunnelservice <name>`. Pendekatan paling clean:

- **Install**: `wireguard.exe /installtunnelservice <path-to-conf>` (creates `WireGuardTunnel$<TunnelName>` Windows service)
- **Start/Stop**: `ServiceController` standar
- **Status query**: `wg show` CLI (lebih mudah parse) atau named-pipe protocol (lebih cepat tapi binary)

```csharp
using System;
using System.Diagnostics;
using System.IO;
using System.Runtime.Versioning;
using System.Security.Principal;
using System.ServiceProcess;
using System.Text;
using System.Threading;
using System.Threading.Tasks;

namespace HermesNetwork.Sase.Supervisor.Windows;

[SupportedOSPlatform("windows")]
public sealed class WindowsTunnelSupervisor : ITunnelSupervisor
{
    public string TunnelName { get; }

    private readonly string _wireguardExe;
    private readonly string _wgExe;
    private readonly string _configPath;

    public WindowsTunnelSupervisor(string tunnelName)
    {
        TunnelName = tunnelName ?? throw new ArgumentNullException(nameof(tunnelName));
        var wgDir = @"C:\Program Files\WireGuard";
        _wireguardExe = Path.Combine(wgDir, "wireguard.exe");
        _wgExe        = Path.Combine(wgDir, "wg.exe");

        // Lokasi config sebelum kita pass ke /installtunnelservice
        _configPath = Path.Combine(
            Environment.GetFolderPath(Environment.SpecialFolder.LocalApplicationData),
            "HermesNetwork360Guard", "Sase", $"{tunnelName}.conf");
    }

    public bool IsElevated
    {
        get
        {
            using var id = WindowsIdentity.GetCurrent();
            return new WindowsPrincipal(id).IsInRole(WindowsBuiltInRole.Administrator);
        }
    }

    private string ServiceName => $"WireGuardTunnel${TunnelName}";

    public async Task ApplyConfigAsync(string wgConfig, CancellationToken ct = default)
    {
        Directory.CreateDirectory(Path.GetDirectoryName(_configPath)!);
        await File.WriteAllTextAsync(_configPath, wgConfig, new UTF8Encoding(false), ct);

        // Restrict ACL: only admin + SYSTEM (config contains private key)
        SetSecureAcl(_configPath);

        // Install service kalau belum ada, kalau sudah ada uninstall + reinstall biar config terbaru di-pakai
        if (await GetStatusAsync(ct) is { State: TunnelState.NotInstalled })
        {
            await InstallServiceAsync(ct);
        }
        else
        {
            await UninstallAsync(ct);
            await InstallServiceAsync(ct);
        }
    }

    public async Task StartAsync(CancellationToken ct = default)
    {
        var status = await GetStatusAsync(ct);
        if (status.State == TunnelState.Up || status.State == TunnelState.Starting) return;
        if (status.State == TunnelState.NotInstalled)
            throw new InvalidOperationException(
                "Tunnel tidak terinstall — panggil ApplyConfigAsync dulu.");

        using var sc = new ServiceController(ServiceName);
        sc.Start();
        await WaitForServiceAsync(sc, ServiceControllerStatus.Running, ct);
    }

    public async Task StopAsync(CancellationToken ct = default)
    {
        var status = await GetStatusAsync(ct);
        if (status.State == TunnelState.Stopped || status.State == TunnelState.NotInstalled) return;

        using var sc = new ServiceController(ServiceName);
        if (sc.Status == ServiceControllerStatus.Running)
        {
            sc.Stop();
            await WaitForServiceAsync(sc, ServiceControllerStatus.Stopped, ct);
        }
    }

    public async Task UninstallAsync(CancellationToken ct = default)
    {
        var psi = new ProcessStartInfo(_wireguardExe,
            $"/uninstalltunnelservice {TunnelName}")
        {
            UseShellExecute = true,
            Verb = IsElevated ? "" : "runas",
            CreateNoWindow = true,
        };
        using var p = Process.Start(psi);
        if (p is not null) await p.WaitForExitAsync(ct);

        if (File.Exists(_configPath))
            File.Delete(_configPath);
    }

    public async Task<TunnelStatus> GetStatusAsync(CancellationToken ct = default)
    {
        // 1. Check service state
        TunnelState state;
        try
        {
            using var sc = new ServiceController(ServiceName);
            state = sc.Status switch
            {
                ServiceControllerStatus.Running      => TunnelState.Up,
                ServiceControllerStatus.Stopped      => TunnelState.Stopped,
                ServiceControllerStatus.StartPending => TunnelState.Starting,
                ServiceControllerStatus.StopPending  => TunnelState.Stopping,
                _                                    => TunnelState.Faulted,
            };
        }
        catch (InvalidOperationException)
        {
            return new TunnelStatus(TunnelState.NotInstalled, null, 0, 0, 0, null, null);
        }

        if (state != TunnelState.Up)
            return new TunnelStatus(state, null, 0, 0, 0, null, null);

        // 2. Tanya wg show untuk detail
        return await ParseWgShowAsync(ct) with { State = state };
    }

    public async Task<DateTimeOffset> WaitHandshakeAsync(
        TimeSpan? timeout = null, CancellationToken ct = default)
    {
        var deadline = DateTime.UtcNow + (timeout ?? TimeSpan.FromSeconds(15));
        while (DateTime.UtcNow < deadline)
        {
            ct.ThrowIfCancellationRequested();
            var st = await GetStatusAsync(ct);
            if (st.LastHandshake is { } hs && hs > DateTimeOffset.UtcNow.AddMinutes(-3))
                return hs;
            await Task.Delay(1_000, ct);
        }
        throw new TimeoutException("WireGuard handshake belum tercapai dalam window timeout.");
    }

    // ---------------- helpers ----------------

    private async Task InstallServiceAsync(CancellationToken ct)
    {
        var psi = new ProcessStartInfo(_wireguardExe,
            $"/installtunnelservice \"{_configPath}\"")
        {
            UseShellExecute = true,
            Verb = IsElevated ? "" : "runas",
            CreateNoWindow = true,
        };
        using var p = Process.Start(psi);
        if (p is null) throw new InvalidOperationException("Could not start wireguard.exe");
        await p.WaitForExitAsync(ct);
        if (p.ExitCode != 0)
            throw new InvalidOperationException(
                $"wireguard.exe /installtunnelservice exited with code {p.ExitCode}");
    }

    private async Task<TunnelStatus> ParseWgShowAsync(CancellationToken ct)
    {
        var (code, stdout, _) = await RunAsync(_wgExe, $"show {TunnelName} dump", ct);
        if (code != 0)
            return new TunnelStatus(TunnelState.Faulted, null, 0, 0, 0, null, null);

        // Format dump: tab-separated columns. Line pertama = interface, sisanya = peers.
        // interface: priv-key   pub-key   listen-port   fwmark
        // peer:      pub-key   psk   endpoint   allowed-ips   latest-handshake   rx-bytes   tx-bytes   keepalive
        long rx = 0, tx = 0;
        DateTimeOffset? lastHs = null;
        int peerCount = 0;
        var lines = stdout.Split('\n', StringSplitOptions.RemoveEmptyEntries);
        for (int i = 1; i < lines.Length; i++)
        {
            var cols = lines[i].Split('\t');
            if (cols.Length < 8) continue;
            peerCount++;
            if (long.TryParse(cols[5], out var rxB)) rx += rxB;
            if (long.TryParse(cols[6], out var txB)) tx += txB;
            if (long.TryParse(cols[4], out var hsUnix) && hsUnix > 0)
            {
                var t = DateTimeOffset.FromUnixTimeSeconds(hsUnix);
                if (lastHs is null || t > lastHs) lastHs = t;
            }
        }

        return new TunnelStatus(TunnelState.Up, lastHs, rx, tx, peerCount, null, null);
    }

    private static async Task WaitForServiceAsync(
        ServiceController sc, ServiceControllerStatus target, CancellationToken ct)
    {
        var deadline = DateTime.UtcNow.AddSeconds(30);
        while (DateTime.UtcNow < deadline)
        {
            ct.ThrowIfCancellationRequested();
            sc.Refresh();
            if (sc.Status == target) return;
            await Task.Delay(500, ct);
        }
        throw new TimeoutException(
            $"Service {sc.ServiceName} tidak mencapai status {target} dalam 30s.");
    }

    private static async Task<(int Code, string Stdout, string Stderr)> RunAsync(
        string file, string args, CancellationToken ct, int timeoutSeconds = 15)
    {
        var psi = new ProcessStartInfo(file, args)
        {
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            CreateNoWindow = true,
        };
        using var p = Process.Start(psi)!;
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(timeoutSeconds));

        var stdoutTask = p.StandardOutput.ReadToEndAsync();
        var stderrTask = p.StandardError.ReadToEndAsync();
        await p.WaitForExitAsync(cts.Token);

        return (p.ExitCode, await stdoutTask, await stderrTask);
    }

    private static void SetSecureAcl(string path)
    {
        // Pakai ACL Windows untuk batasi akses ke admin + SYSTEM
        // (Implementasi penuh: System.Security.AccessControl)
        // Stub: rely on default ACL inherited dari %LOCALAPPDATA% user
        // Untuk produksi, implementasikan dengan FileSecurity + AddAccessRule
    }
}
```

## 4.5 Implementasi macOS

Pakai `wg-quick` (script wrapper di atas `wg`) plus LaunchDaemon untuk persistent service.

```csharp
using System;
using System.Diagnostics;
using System.IO;
using System.Runtime.Versioning;
using System.Threading;
using System.Threading.Tasks;

namespace HermesNetwork.Sase.Supervisor.Mac;

[SupportedOSPlatform("macos")]
public sealed class MacTunnelSupervisor : ITunnelSupervisor
{
    public string TunnelName { get; }
    private readonly string _configPath;       // /etc/wireguard/<name>.conf
    private readonly string _plistPath;        // /Library/LaunchDaemons/com.hermesnetwork.sase.plist
    private readonly string _label;            // com.hermesnetwork.sase

    public MacTunnelSupervisor(string tunnelName)
    {
        TunnelName = tunnelName;
        _configPath = $"/etc/wireguard/{tunnelName}.conf";
        _label = "com.hermesnetwork.sase";
        _plistPath = $"/Library/LaunchDaemons/{_label}.plist";
    }

    public bool IsElevated => Environment.UserName == "root";

    public async Task ApplyConfigAsync(string wgConfig, CancellationToken ct = default)
    {
        // Tulis config ke /tmp dulu, lalu pindah dengan elevation
        var tmp = Path.Combine(Path.GetTempPath(), $"sase-{Guid.NewGuid():N}.conf");
        await File.WriteAllTextAsync(tmp, wgConfig, ct);

        var cmd = $"mkdir -p /etc/wireguard && " +
                  $"mv \\\"{tmp}\\\" \\\"{_configPath}\\\" && " +
                  $"chmod 600 \\\"{_configPath}\\\" && " +
                  $"chown root:wheel \\\"{_configPath}\\\"";
        await RunWithElevationAsync(cmd, ct);

        // Install LaunchDaemon kalau belum ada
        if (!File.Exists(_plistPath))
            await InstallLaunchDaemonAsync(ct);
    }

    public Task StartAsync(CancellationToken ct = default)
        => RunWithElevationAsync($"launchctl bootstrap system {_plistPath} 2>/dev/null; " +
                                 $"launchctl kickstart -k system/{_label}", ct);

    public Task StopAsync(CancellationToken ct = default)
        => RunWithElevationAsync($"launchctl bootout system/{_label} 2>/dev/null; " +
                                 $"wg-quick down {TunnelName} 2>/dev/null", ct);

    public async Task UninstallAsync(CancellationToken ct = default)
    {
        try { await StopAsync(ct); } catch { }
        await RunWithElevationAsync(
            $"rm -f \\\"{_plistPath}\\\" \\\"{_configPath}\\\"", ct);
    }

    public async Task<TunnelStatus> GetStatusAsync(CancellationToken ct = default)
    {
        if (!File.Exists(_configPath))
            return new TunnelStatus(TunnelState.NotInstalled, null, 0, 0, 0, null, null);

        var (code, stdout, _) = await RunAsync("wg", $"show {TunnelName} dump", ct);
        if (code != 0)
        {
            // Tunnel tidak up
            return new TunnelStatus(TunnelState.Stopped, null, 0, 0, 0, null, null);
        }

        // Same parsing as Windows
        long rx = 0, tx = 0;
        DateTimeOffset? lastHs = null;
        int peerCount = 0;
        var lines = stdout.Split('\n', StringSplitOptions.RemoveEmptyEntries);
        for (int i = 1; i < lines.Length; i++)
        {
            var cols = lines[i].Split('\t');
            if (cols.Length < 8) continue;
            peerCount++;
            if (long.TryParse(cols[5], out var rxB)) rx += rxB;
            if (long.TryParse(cols[6], out var txB)) tx += txB;
            if (long.TryParse(cols[4], out var hsUnix) && hsUnix > 0)
            {
                var t = DateTimeOffset.FromUnixTimeSeconds(hsUnix);
                if (lastHs is null || t > lastHs) lastHs = t;
            }
        }
        return new TunnelStatus(TunnelState.Up, lastHs, rx, tx, peerCount, null, null);
    }

    public async Task<DateTimeOffset> WaitHandshakeAsync(
        TimeSpan? timeout = null, CancellationToken ct = default)
    {
        var deadline = DateTime.UtcNow + (timeout ?? TimeSpan.FromSeconds(15));
        while (DateTime.UtcNow < deadline)
        {
            ct.ThrowIfCancellationRequested();
            var st = await GetStatusAsync(ct);
            if (st.LastHandshake is { } hs && hs > DateTimeOffset.UtcNow.AddMinutes(-3))
                return hs;
            await Task.Delay(1_000, ct);
        }
        throw new TimeoutException("WireGuard handshake belum tercapai dalam window timeout.");
    }

    // ---------------- helpers ----------------

    private async Task InstallLaunchDaemonAsync(CancellationToken ct)
    {
        var plist = $"""
            <?xml version="1.0" encoding="UTF-8"?>
            <!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
            <plist version="1.0">
            <dict>
                <key>Label</key>
                <string>{_label}</string>
                <key>ProgramArguments</key>
                <array>
                    <string>/usr/local/bin/wg-quick</string>
                    <string>up</string>
                    <string>{TunnelName}</string>
                </array>
                <key>RunAtLoad</key><true/>
                <key>KeepAlive</key><false/>
                <key>StandardOutPath</key><string>/var/log/sase-tunnel.out.log</string>
                <key>StandardErrorPath</key><string>/var/log/sase-tunnel.err.log</string>
                <key>UserName</key><string>root</string>
            </dict>
            </plist>
            """;

        var tmp = Path.Combine(Path.GetTempPath(), $"sase-plist-{Guid.NewGuid():N}.plist");
        await File.WriteAllTextAsync(tmp, plist, ct);

        await RunWithElevationAsync(
            $"mv \\\"{tmp}\\\" \\\"{_plistPath}\\\" && " +
            $"chmod 644 \\\"{_plistPath}\\\" && " +
            $"chown root:wheel \\\"{_plistPath}\\\"", ct);
    }

    private async Task RunWithElevationAsync(
        string shellCmd, CancellationToken ct, int timeoutSeconds = 60)
    {
        var escaped = shellCmd.Replace("\"", "\\\"");
        var script = $"do shell script \"{escaped}\" with administrator privileges";

        var (code, _, stderr) = await RunAsync("osascript", $"-e '{script}'", ct, timeoutSeconds);
        if (code != 0)
            throw new InvalidOperationException(
                $"Elevated command failed (exit {code}): {stderr}");
    }

    private static async Task<(int Code, string Stdout, string Stderr)> RunAsync(
        string file, string args, CancellationToken ct, int timeoutSeconds = 30)
    {
        var psi = new ProcessStartInfo(file, args)
        {
            RedirectStandardOutput = true,
            RedirectStandardError = true,
            UseShellExecute = false,
            CreateNoWindow = true,
        };
        using var p = Process.Start(psi)!;
        using var cts = CancellationTokenSource.CreateLinkedTokenSource(ct);
        cts.CancelAfter(TimeSpan.FromSeconds(timeoutSeconds));
        var so = p.StandardOutput.ReadToEndAsync();
        var se = p.StandardError.ReadToEndAsync();
        await p.WaitForExitAsync(cts.Token);
        return (p.ExitCode, await so, await se);
    }
}
```

## 4.6 Penggunaan dari ConnectionService

```csharp
public sealed class SaseConnectionService
{
    private readonly ITunnelSupervisor _supervisor;
    private readonly ISaseConfigClient _config;
    private readonly ILogger<SaseConnectionService> _log;

    public async Task<ConnectionResult> ConnectAsync(CancellationToken ct = default)
    {
        // 1. Get latest config
        var configDto = await _config.RequestConfigAsync(ct);
        var wgConfig = WgConfigBuilder.Build(configDto);

        // 2. Apply
        await _supervisor.ApplyConfigAsync(wgConfig, ct);

        // 3. Start
        await _supervisor.StartAsync(ct);

        // 4. Wait for handshake
        try
        {
            var hsTime = await _supervisor.WaitHandshakeAsync(TimeSpan.FromSeconds(15), ct);
            _log.LogInformation("SASE handshake at {Time}", hsTime);
            return ConnectionResult.Connected(hsTime);
        }
        catch (TimeoutException)
        {
            _log.LogWarning("SASE handshake timeout — tunnel up tapi tidak ada peer reply");
            return ConnectionResult.HandshakeTimeout;
        }
    }

    public Task DisconnectAsync(CancellationToken ct = default)
        => _supervisor.StopAsync(ct);

    public async Task<TunnelStatus> GetStatusAsync(CancellationToken ct = default)
        => await _supervisor.GetStatusAsync(ct);
}
```

## 4.7 WgConfigBuilder helper

Helper untuk merangkai INI WireGuard dari `SaseConfigDto`:

```csharp
using System.Text;

namespace HermesNetwork.Sase;

public static class WgConfigBuilder
{
    public static string Build(SaseConfigDto dto)
    {
        var sb = new StringBuilder();
        sb.AppendLine("[Interface]");
        sb.AppendLine($"PrivateKey = {dto.PrivateKey}");
        sb.AppendLine($"Address = {dto.InterfaceAddress}");
        if (!string.IsNullOrWhiteSpace(dto.Dns))
            sb.AppendLine($"DNS = {dto.Dns}");
        if (dto.Mtu is int mtu)
            sb.AppendLine($"MTU = {mtu}");
        sb.AppendLine();

        foreach (var peer in dto.Peers)
        {
            sb.AppendLine("[Peer]");
            sb.AppendLine($"PublicKey = {peer.PublicKey}");
            if (!string.IsNullOrWhiteSpace(peer.PresharedKey))
                sb.AppendLine($"PresharedKey = {peer.PresharedKey}");
            sb.AppendLine($"Endpoint = {peer.Endpoint}");
            sb.AppendLine($"AllowedIPs = {peer.AllowedIPs}");
            sb.AppendLine($"PersistentKeepalive = {peer.KeepaliveSeconds}");
            sb.AppendLine();
        }
        return sb.ToString();
    }
}
```

## 4.8 Testing

### 4.8.1 Mock supervisor untuk unit test

```csharp
public sealed class FakeTunnelSupervisor : ITunnelSupervisor
{
    public string TunnelName => "test";
    public bool IsElevated => true;
    public TunnelStatus CurrentStatus { get; set; } = new(TunnelState.Stopped, null, 0, 0, 0, null, null);
    public string? AppliedConfig { get; private set; }

    public Task ApplyConfigAsync(string c, CancellationToken ct = default)
    { AppliedConfig = c; return Task.CompletedTask; }
    public Task StartAsync(CancellationToken ct = default)
    { CurrentStatus = CurrentStatus with { State = TunnelState.Up,
        LastHandshake = DateTimeOffset.UtcNow }; return Task.CompletedTask; }
    public Task StopAsync(CancellationToken ct = default)
    { CurrentStatus = CurrentStatus with { State = TunnelState.Stopped }; return Task.CompletedTask; }
    public Task UninstallAsync(CancellationToken ct = default)
    { CurrentStatus = new(TunnelState.NotInstalled, null, 0, 0, 0, null, null); return Task.CompletedTask; }
    public Task<TunnelStatus> GetStatusAsync(CancellationToken ct = default)
        => Task.FromResult(CurrentStatus);
    public Task<DateTimeOffset> WaitHandshakeAsync(TimeSpan? t = null, CancellationToken ct = default)
        => Task.FromResult(CurrentStatus.LastHandshake ?? DateTimeOffset.UtcNow);
}
```

### 4.8.2 Manual integration test

**Windows:**
```powershell
# Generate test config dengan key real (manual register peer di gateway)
$config = @"
[Interface]
PrivateKey = $(wg genkey)
Address = 10.99.0.5/32
DNS = 10.0.0.1

[Peer]
PublicKey = <gateway-pub-key>
Endpoint = n1.ndr24.com:51820
AllowedIPs = 10.0.0.0/8
PersistentKeepalive = 25
"@

# Test via supervisor
dotnet run --project HermesNetwork.SupervisorTest -- apply "$config"
dotnet run --project HermesNetwork.SupervisorTest -- start
& "C:\Program Files\WireGuard\wg.exe" show
dotnet run --project HermesNetwork.SupervisorTest -- stop
dotnet run --project HermesNetwork.SupervisorTest -- uninstall
```

## 4.9 Best practices

- **Selalu cek `IsElevated`** sebelum `ApplyConfig` / `Install` / `Uninstall` — minta UAC kalau false
- **Timeout** semua external process (`wireguard.exe`, `wg-quick`, `osascript`) dengan `CancellationTokenSource.CancelAfter`
- **Jangan log full config WireGuard** — mengandung PrivateKey, PSK
- **Set ACL `0o600` (Mac) / restricted ACL (Windows)** pada file config — hanya admin/root yang bisa baca
- **Idempotent** semua operasi — `StartAsync` di tunnel yang sudah Up = no-op
- **Handle "tunnel up tapi tidak ada handshake" graceful** — bisa terjadi saat NAT block atau gateway down

---

[← Bab 3 Prasyarat]({{ site.baseurl }}{% link docs/03-prasyarat.md %}){: .btn }
[Bab 5 — Config Service →]({{ site.baseurl }}{% link docs/05-config-service.md %}){: .btn .btn-primary }
