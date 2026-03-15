# Defining a Server

A **Server** in LuciferCore is a class that extends `WssServer` (or `HttpsServer`) and is decorated with the `[Server]` attribute. LuciferCore auto-discovers and instantiates it at startup.

---

## The `[Server]` Attribute

```csharp
[Server("ChatServer", 8443)]
public class ChatServer : WssServer { ... }
```

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | A human-readable identifier for this server |
| `port` | `int` | The port the server listens on |

---

## Binding Configuration with `[Config]`

Use `[Config]` on static properties to bind values from your environment or configuration source — no hardcoding required.

```csharp
[Config("WWW", "assets/client/dev")]
private static string _staticContentPath { get; set; } = string.Empty;

[Config("CERTIFICATE", "assets/tools/certificates/server.pfx")]
private static string s_certPath { get; set; } = string.Empty;

[Config("CERT_PASSWORD", "RootCA!SecureKey@Example2025Strong")]
private static string s_certPassword { get; set; } = string.Empty;
```

| Parameter | Description |
|---|---|
| Key | The configuration key to look up |
| Default value | Fallback if the key is not found |

---

## Static Content & URL Mapping

Serve static files and define URL-to-file mappings directly in the constructor:

```csharp
AddStaticContent(_staticContentPath);
Cache.Freeze();

Mapping = new(true)
{
    { "/",          "/index.html"     },
    { "/dashboard", "/dashboard.html" }
};
Mapping.Freeze();
```

Calling `Cache.Freeze()` and `Mapping.Freeze()` locks these structures, making them read-only and safe for concurrent access.

---

## SSL Context

### Development (Auto-generated)

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

In `DEBUG` builds, `CreateDevelopmentContext()` generates a self-signed certificate automatically. In `RELEASE` builds, a real `.pfx` certificate is loaded.

---

## Full Example

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
            { "/",          "/index.html"     },
            { "/404",       "/404.html"       },
            { "/login",     "/login.html"     },
            { "/dashboard", "/dashboard.html" }
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
