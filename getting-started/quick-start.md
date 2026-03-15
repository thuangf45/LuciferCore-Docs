# Quick Start

This page gets you from zero to a running WSS + HTTPS server in **5 steps** — no configuration files, no boilerplate, just attributes.

---

## Step 1 — Define Your Server

Decorate your class with `[Server]` to register it as a high-performance server instance. Use `[Config]` to bind configurable values (paths, certificates, etc.) without hardcoding.

```csharp
[Server("ChatServer", 8443)]
public class ChatServer : WssServer
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public ChatServer(SslContext context, IPAddress address, int port) : base(context, address, port)
    {
        AddStaticContent(_staticContentPath);
        Cache.Freeze();

        Mapping = new(true)
        {
            { "/",          "/index.html"      },
            { "/404",       "/404.html"        },
            { "/home",      "/home.html"       },
            { "/login",     "/login.html"      },
            { "/register",  "/register.html"   },
            { "/dashboard", "/dashboard.html"  },
            { "/search",    "/search.html"     }
        };
        Mapping.Freeze();
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public ChatServer(int port) : this(CreateSslContext(), IPAddress.Any, port) { }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected override ChatSession CreateSession() => new(this);

    [Config("WWW", "assets/client/dev")]
    private static string _staticContentPath { get; set; } = string.Empty;

    [Config("CERTIFICATE", "assets/tools/certificates/server.pfx")]
    private static string s_certPath { get; set; } = string.Empty;

    [Config("CERT_PASSWORD", "RootCA!SecureKey@Example2025Strong")]
    private static string s_certPassword { get; set; } = string.Empty;

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public static SslContext CreateSslContext()
    {
#if DEBUG
        return SslContext.CreateDevelopmentContext();
#else
        var cert = X509CertificateLoader.LoadPkcs12FromFile(s_certPath, s_certPassword);
        return new(SslProtocols.Tls12, cert);
#endif
    }
}
```

> In `DEBUG` mode, `CreateSslContext()` automatically generates a development certificate — no setup needed.

---

## Step 2 — Configure Your Session

Apply `[RateLimiter]` at the session level and forward incoming messages to `Lucifer.Dispatch()`.

```csharp
[RateLimiter(10, 1)]
public partial class ChatSession : WssSession
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public ChatSession(ChatServer server) : base(server) { }

    // Forward binary WebSocket messages to the dispatcher
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected override void OnWsReceived(byte[] buffer, long offset, long size)
        => Lucifer.Dispatch(this, buffer, offset, size);

    // Forward HTTP requests to the dispatcher
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected override void OnReceivedRequest(RequestModel request)
        => Lucifer.Dispatch(this, request);
}
```

---

## Step 3 — Implement Your Handlers

### WebSocket Handler

```csharp
[Handler("v1", "wss")]
internal class WssHandler : WssHandlerBase
{
    public ConcurrentQueue<(byte[], long, long)> Messages = new();

    [WsMessage("GetMessage")]
    [Safe("")]
    [RateLimiter(10, 1)]
    [Authorize(UserRole.Guest)]
    public void GetMessage([Session] ChatSession session, [Data] PacketModel data)
    {
        foreach (var (buffer, offset, length) in Messages)
            session.SendBinaryAsync(buffer.AsSpan((int)offset, (int)length));
    }

    [WsMessage("ChatMessage")]
    [Safe("")]
    [RateLimiter(20, 1)]
    [Authorize(UserRole.Guest)]
    public void SendChat([Session] ChatSession session, [Data] PacketModel data)
    {
        using var _ = data;
        ((WssServer)session.Server).MulticastBinary(data.Buffer);
    }

    [WsMessage("Default")]
    [Safe("")]
    [RateLimiter(20, 1)]
    [Authorize(UserRole.Guest)]
    public void Default([Session] ChatSession session, [Data] PacketModel data)
        => throw new NotImplementedException();
}
```

### HTTP Handler

```csharp
[Handler("v1", "/api/user")]
internal class HttpsHandler : HttpsHandlerBase
{
    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpHead("")]
    protected void HeadHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpGet("")]
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpPost("")]
    protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpPut("")]
    protected void PutHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpDelete("")]
    protected void DeleteHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpOptions("")]
    protected void OptionsHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpTrace("")]
    protected void TraceHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();
}
```

---

## Step 4 — Add a Background Manager

```csharp
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase
{
    protected override void Setup()
    {
        TimeDelay  = 1000;
        ErrorDelay = 1000;
    }

    protected override void Update()
    {
        Lucifer.Log(this, "Master is running....");
    }

    protected override void Dispose(bool disposing)
    {
        base.Dispose(disposing);
    }
}
```

---

## Step 5 — Run

```csharp
using LuciferCore.Main;

Lucifer.Run(); // auto-discovers all [Server], [Handler], [Manager]
```

That's it. LuciferCore auto-discovers every decorated class and starts the full ecosystem.

---

## Next Steps

- Learn about each component in depth in the [Guides](../guides/defining-a-server.md) section.
- Customize the entry point with [Console Commands](../guides/console-commands.md).
