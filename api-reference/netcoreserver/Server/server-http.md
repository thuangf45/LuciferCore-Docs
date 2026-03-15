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
| `Cache` | `FileCache` | Static content cache shared across all sessions. Call `Cache.Freeze()` after loading to lock into read-only mode |
| `Mapping` | `Utf8Map<ByteString>` | URL path → static file path mapping. Consulted first by the static path resolver before the cache |

---

## Static Content

```csharp
void AddStaticContent(string path, string prefix = "/", string filter = "*.*", TimeSpan? timeout = null)
void RemoveStaticContent(string path)
void ClearStaticContent()
```

`AddStaticContent` scans `path` recursively, builds pre-assembled HTTP `200 OK` responses for each file (including `Content-Type` and `Cache-Control` headers), and inserts them into `Cache`. Default timeout is **1 hour**. A `FileSystemWatcher` auto-reloads changed files.

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

All server lifecycle, socket options, session management, and TCP multicast methods are inherited from `TcpServer` and `ServerTransport`. See [ServerTransport](transport-server.md).

---

## Remarks

- `Cache` is disposed automatically when the server is disposed.
- `WsServer` extends `HttpServer` — the same `Cache` and `Mapping` are available on WebSocket servers.
