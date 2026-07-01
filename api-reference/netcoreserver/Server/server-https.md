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
| `Cache` | `FileCache` | Static content cache shared across all sessions. Disposed automatically when the server is disposed |
| `Mapping` | `Utf8Map<ByteString>` | URL path → static file path mapping. Consulted first by `GetStaticPath` before the cache |

---

## Static Content

```csharp
void AddStaticContent(string path, string prefix = "/", string filter = "*.*", TimeSpan? timeout = null)
void RemoveStaticContent(string path)
void ClearStaticContent()
```

`AddStaticContent` ensures system default files (`index.html`, `404.html`) exist in `Mapping`/`Cache`, then scans `path` recursively and inserts every file found into `Cache` — **note:** the initial scan loads *all* files in the directory; `filter` is only applied later to the `FileSystemWatcher` used for auto-reload, not to the initial load. Each cached file is stored as a pre-assembled HTTP `200 OK` response with `Content-Type` and `Cache-Control` headers set. Default `timeout` is **1 hour**.

Calling `AddStaticContent` again with the same `path` re-registers it: existing entries for that path are removed first, then reloaded from scratch.

Behavior identical to `HttpServer.AddStaticContent`.

```csharp
void RemoveStaticContent(string path)
```
Removes a previously added directory (and its entries/watcher) from `Cache`.

```csharp
void ClearStaticContent()
```
Clears the entire `Cache`.

---

## Request Path Resolution

```csharp
protected internal virtual ReadOnlySpan<byte> GetStaticPath(RequestModel request)
```

Resolves the file path to serve for an incoming request, in order:

1. **Mapping lookup** — if the request URL (query string stripped) matches a key in `Mapping`, return the mapped path.
2. **`.html` fallback** — if the URL ends with `.html` (case-insensitive) and wasn't in `Mapping`, treat it as missing and return the `404` mapping (`Mapping[StorageData.Key404]`).
3. **Cache lookup** — if the path exists in `Cache`, return the path as-is.
4. **Not found** — otherwise return the `404` mapping.

If the URL is empty, `/` is used. Override this method to customize routing logic in derived servers.

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

SSL configuration (`Context`, protocol, certificate) is inherited from `SslServer`. All server lifecycle and session management methods are inherited from `ServerTransport`.

---

## Remarks

- `Cache` and its `FileSystemWatcher`s are owned by `FileCache`, not `HttpsServer` — see the `FileCache` docs for `Freeze()`, watcher-based auto-reload, and expiry behavior.
- `WssServer` extends `HttpsServer` — the same `Cache` and `Mapping` are available on secure WebSocket servers.
- `Dispose(bool)` calls `Cache.Dispose()` when disposing managed resources.

---