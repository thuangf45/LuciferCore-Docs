# HttpServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

Plain HTTP server. Extends `TcpServer` with HTTP request parsing, static content caching, and URL mapping.

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
protected override HttpSession CreateSession()
```

Override to return a custom session type:

```csharp
public class MyServer : HttpServer
{
    public MyServer(int port) : base(IPAddress.Any, port) { }

    protected override HttpSession CreateSession() => new MySession(this);
}
```

---

## Inherited API

All server lifecycle, socket options, session management, and TCP multicast methods are inherited from `TcpServer` and `ServerTransport`.

---

## Remarks

- `Cache` and its `FileSystemWatcher`s are owned by `FileCache`, not `HttpServer` — see the `FileCache` docs for `Freeze()`, watcher-based auto-reload, and expiry behavior.
- `WsServer` extends `HttpServer` — the same `Cache` and `Mapping` are available on WebSocket servers.
- `Dispose(bool)` calls `Cache.Dispose()` when disposing managed resources.