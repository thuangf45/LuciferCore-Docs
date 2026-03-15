# HttpsServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

Secure HTTPS server. Extends `SslServer` with HTTP request parsing, static content caching, and URL mapping over TLS.

```csharp
public class HttpsServer : SslServer
```

---

## Constructors

```csharp
public HttpsServer(SslContext context, IPAddress address, int port)
public HttpsServer(SslContext context, string address, int port)
public HttpsServer(SslContext context, IPEndPoint endpoint)
public HttpsServer(SslContext context, DnsEndPoint endpoint)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Cache` | `FileCache` | Static content cache shared across all sessions. Call `Cache.Freeze()` after loading |
| `Mapping` | `Utf8Map<ByteString>` | URL path → static file path mapping |

---

## Static Content

```csharp
void AddStaticContent(string path, string prefix = "/", string filter = "*.*", TimeSpan? timeout = null)
void RemoveStaticContent(string path)
void ClearStaticContent()
```

Identical behavior to `HttpServer.AddStaticContent`. See [HttpServer](server-http.md).

---

## Session Factory

```csharp
protected override HttpsSession CreateSession()
```

Override to return a custom session type:

```csharp
[Server("ChatServer", 8443)]
public class ChatServer : HttpsServer
{
    public ChatServer(SslContext context, IPAddress address, int port)
        : base(context, address, port) { }

    protected override ChatSession CreateSession() => new(this);
}
```

---

## Inherited API

SSL configuration (`Context`, protocol, certificate) is inherited from `SslServer`. See [SslServer](ssl-server.md). All server lifecycle and session management methods are inherited from `ServerTransport`. See [ServerTransport](transport-server.md).

---

## Remarks

- `Cache` is disposed automatically when the server is disposed.
- `WssServer` extends `HttpsServer` — the same `Cache` and `Mapping` are available on secure WebSocket servers.
- Use `SslContext.CreateDevelopmentContext()` in `DEBUG` builds to avoid needing a real certificate. See [SslContext](ssl-context.md).
