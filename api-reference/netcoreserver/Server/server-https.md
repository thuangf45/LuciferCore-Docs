# HttpsServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

`HttpsServer` is a secure HTTP server class.  
It extends `SslServer` and adds HTTP parsing, static file cache, and URL mapping over TLS.

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
| `Cache` | `FileCache` | Shared static file cache |
| `Mapping` | `Utf8Map<ByteString>` | URL path → static file path map |

---

## Static content API

```csharp
void AddStaticContent(string path, string prefix = "/", string filter = "*.*", TimeSpan? timeout = null)
void RemoveStaticContent(string path)
void ClearStaticContent()
```

### `AddStaticContent(...)`

- Adds a folder to static hosting
- Ensures default files like `index.html` and `404.html`
- Loads files into `Cache`
- Default `timeout` is 1 hour
- If called again with same `path`, old entries are removed and reloaded

Behavior is the same as `HttpServer.AddStaticContent(...)`.

### `RemoveStaticContent(path)`

- Removes one registered static folder and its cached entries

### `ClearStaticContent()`

- Clears all static cache entries

---

## Request path resolution

```csharp
protected internal virtual ReadOnlySpan<byte> GetStaticPath(RequestModel request)
```

Path is resolved in this order:

1. check `Mapping`
2. if URL ends with `.html` and not mapped, use `404`
3. check `Cache`
4. fallback to `404`

If URL is empty, `/` is used.

Override this method to implement custom routing.

---

## Custom session type

```csharp
protected override HttpsSession CreateSession()
```

Example:

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

`HttpsServer` adds HTTPS + HTTP/static features.  
Other APIs are inherited from `SslServer` / `ServerTransport`, including:

- SSL context/certificate handling
- `Start()`, `Stop()`, `Restart()`
- session management
- lifecycle hooks/events
- `Dispose()`

---

## Notes

- `Cache` is shared by all HTTPS sessions.
- `WssServer` extends `HttpsServer`, so it can reuse `Cache` and `Mapping`.
- On dispose, `HttpsServer` also disposes `Cache`.