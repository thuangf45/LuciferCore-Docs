# HttpServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

`HttpServer` is a plain HTTP server class.  
It extends `TcpServer` and adds HTTP parsing, static file cache, and URL mapping.

```csharp
public class HttpServer : TcpServer
```

---

## Constructors

```csharp
public HttpServer(IPAddress address, int port)
public HttpServer(string address, int port)
public HttpServer(IPEndPoint endpoint)
public HttpServer(DnsEndPoint endpoint)
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
protected override HttpSession CreateSession()
```

Example:

```csharp
public class MyServer : HttpServer
{
    public MyServer(int port) : base(IPAddress.Any, port) { }

    protected override HttpSession CreateSession() => new MySession(this);
}
```

---

## Inherited API

`HttpServer` adds HTTP/static features.  
Server lifecycle and session management are inherited from `TcpServer` / `ServerTransport`:

- `Start()`, `Stop()`, `Restart()`
- session management
- multicast
- lifecycle hooks/events
- `Dispose()`

---

## Notes

- `Cache` is shared by all HTTP sessions.
- `WsServer` extends `HttpServer`, so it can reuse `Cache` and `Mapping`.
- On dispose, `HttpServer` also disposes `Cache`.