# Quick Start

Start a WSS + HTTPS server in **6 steps**.  
No config files. No heavy setup. Just attributes.

---

## Step 1) Create a server

Use `[Server]` to register your server.  
Use `[Config]` for values like static folder and certificate path.

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
            { "/", "/index.html" },
            { "/404", "/404.html" }
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
        return new(SslContext.SslProtocols.Tls12, cert);
#endif
    }
}
```

> In `DEBUG`, `CreateSslContext()` uses a development certificate automatically.

---

## Step 2) Create a session

Use `[RateLimiter]` to limit request rate per session.  
Forward HTTP requests to the router with `RouteHandler.Route(...)`.

```csharp
[RateLimiter(10, 1)]
public partial class ChatSession : WssSession
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public ChatSession(ChatServer server) : base(server) { }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected override void OnWsReceived(byte[] buffer, long offset, long size)
    {
        // Handle websocket message
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected override void OnReceivedRequest(RequestModel request)
        => RouteHandler.Route(request, this);
}
```

---

## Step 3) Add route handlers

Use `[Handler]` to define an API group.  
Use HTTP attributes (`[HttpGet]`, `[HttpPost]`, etc.) for each endpoint.

```csharp
[Handler("v1", "/api/user")]
internal class HttpsHandler : RouteHandler
{
    [Authorize(UserRole.Guest)]
    [HttpGet("")]
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session)
    {
        // Handle GET /v1/api/user
    }

    [Authorize(UserRole.Guest)]
    [HttpPost("")]
    protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session)
    {
        // Handle POST /v1/api/user
    }
}
```

---

## Step 4) Add middleware (cross-cutting logic)

Use middleware for auth checks, logging, validation, etc.

```csharp
[Middleware("AuthGuard")]
internal sealed class AuthGuardMiddleware : MiddlewareHandler
{
    protected override bool Handle(IRoutable data, SessionTransport session)
    {
        // false = stop pipeline, true = continue
        return session.IsAuthenticated;
    }
}
```

Attach middleware to a route:

```csharp
[HttpGet("")]
[UseMiddleware("AuthGuard")]
[Authorize(UserRole.Guest)]
public void SendChat([Session] ChatSession session, [Data] PacketModel data)
{
    // Route logic
}
```

---

## Step 5) Add a manager (background logic)

Use a manager for long-running system/background work.

```csharp
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase
{
    protected override void Setup()
    {
        // Initial setup
    }

    protected override void Update()
    {
        Lucifer.Log(this, "Master is running...");
        workload = 10; // remaining jobs
    }
}
```

---

## Step 6) Run the system

```csharp
using LuciferCore.Main;

Lucifer.CMD("/run"u8);
```

LuciferCore auto-discovers classes with attributes like:
`[Server]`, `[Handler]`, `[Manager]`, `[Middleware]`.

You are now running a full event-driven ecosystem.