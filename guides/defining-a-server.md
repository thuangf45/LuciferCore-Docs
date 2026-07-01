# Defining a Server

In LuciferCore, a server is any class that:

1. Inherits a supported server base class
2. Uses the `[Server]` attribute

LuciferCore auto-discovers and creates server instances at startup.

---

## LuciferCore supports many server types

LuciferCore is not limited to only HTTP/WS servers.

You can build servers on top of different protocol layers, for example:
- `TcpServer`
- `SslServer`
- `HttpServer` / `HttpsServer`
- `WsServer` / `WssServer`
- and future protocol servers added by the ecosystem

> To choose the right base class, see the API Reference for server/session types (NetCoreServer-based hierarchy).

---

## Layered transport model

LuciferCore follows a layered transport model.  
Common stacks include:

- `Transport -> TCP -> HTTP -> WS`
- `Transport -> SSL -> HTTPS -> WSS`

Higher layers can serve multiple protocol phases:
- A WS/WSS endpoint can still process HTTP/HTTPS traffic before WebSocket handshake.
- You can hook packets/events at lower layers (TCP/SSL) or higher layers (HTTP/WS), based on your needs.

This gives you one server pipeline that can handle mixed traffic and protocol transitions cleanly.

---

## `[Server]` attribute

```csharp
[Server("ChatServer", 8443)]
public class ChatServer : WssServer { ... }
```

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Server name (display/identification) |
| `port` | `int` | Listening port |

---

## Bind config with `[Config]`

Use `[Config]` on static properties to avoid hardcoded values.

```csharp
[Config("WWW", "assets/client/dev")]
private static string _staticContentPath { get; set; } = string.Empty;

[Config("CERTIFICATE", "assets/tools/certificates/server.pfx")]
private static string s_certPath { get; set; } = string.Empty;

[Config("CERT_PASSWORD", "RootCA!SecureKey@Example2025Strong")]
private static string s_certPassword { get; set; } = string.Empty;
```

| Parameter | Meaning |
|---|---|
| Key | Config key |
| Default value | Used if key is missing |

---

## Static files and URL mapping (HTTP-capable layers)

If your chosen base class supports HTTP/HTTPS behavior, you can configure static content and mappings:

```csharp
AddStaticContent(_staticContentPath);
Cache.Freeze();

Mapping = new(true)
{
    { "/", "/index.html" },
    { "/dashboard", "/dashboard.html" }
};
Mapping.Freeze();
```

`Freeze()` makes structures read-only and safer for concurrent access.

---

## SSL context (for SSL/HTTPS/WSS stacks)

```csharp
public static SslContext CreateSslContext()
{
#if DEBUG
    return SslContext.CreateDevelopmentContext();
#else
    var cert = X509CertificateLoader.LoadPkcs12FromFile(s_certPath, s_certPassword);
    return new(SslProtocols.Tls12, cert);
#endif
}
```

- `DEBUG`: development certificate
- `RELEASE`: real `.pfx` certificate

---

## Example (WSS server)

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
        return new(SslProtocols.Tls12, cert);
#endif
    }
}
```

> This is one example only. Check API Reference for other server base classes and protocol stacks.

---