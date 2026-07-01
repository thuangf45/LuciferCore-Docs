# HttpSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

Plain HTTP session. Extends `TcpSession` — handles incremental HTTP request parsing, static content serving, and response sending.

```csharp
public class HttpSession : TcpSession
```

---

## Constructor

```csharp
public HttpSession(HttpServer server)
```

Not instantiated directly — returned by `HttpServer.CreateSession()`. Captures a reference to the server's `Cache` at construction time.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Cache` | `FileCache` | Reference to the server's static content cache (captured in constructor) |
| `Mapping` | `Utf8Map<ByteString>` | Reads through to the parent `HttpServer.Mapping` on each access |

---

## Send Response API

```csharp
long SendResponse(ResponseModel response)
bool SendResponseAsync(ResponseModel response)
```

`SendResponseAsync` enqueues into the async send path and returns immediately — prefer it on hot paths. (Both delegate to the generic `Send<byte>`/`SendAsync<byte>` inherited from `TcpSession`.)

---

## Lifecycle Hooks

Override these in your session subclass:

| Method | When called |
|---|---|
| `OnReceivedRequestHeader(RequestModel)` | Request headers parsed — body may still be pending. Default implementation is empty |
| `OnReceivedRequest(RequestModel)` | **Primary hook** — full request ready and not served from cache. Default implementation is empty; override this to handle the request |
| `OnReceivedRequestError(RequestModel, string)` | Parse error — session is disconnected immediately after |
| `OnReceivedCachedRequest(RequestModel, byte[])` | Static file cache hit — default implementation sends the cached bytes via `SendAsync`. Override to intercept |

To customize **which** requests are treated as static/cacheable, override `GetStaticPath(RequestModel)` on your `HttpServer` subclass, not on `HttpSession`.

---

## Request Dispatch Flow

```csharp
protected internal virtual void OnReceivedRequestInternal(RequestModel request)
```

Called once a full request is parsed. Behavior:

1. If `Lucifer.Allow(this)` is `false` or `Lucifer.Overloaded` is `true`, the request is dropped (silently returns — the `Disconnect()` call is commented out in the current implementation).
2. If the method is `GET` **and** the URL does not contain `/api` (case-insensitive), the session asks the parent `HttpServer` for `GetStaticPath(request)` and checks `Cache` for that path.
   - **Hit** → calls `OnReceivedCachedRequest(request, bytes)` and returns (cached response sent, `OnReceivedRequest` is *not* called).
   - **Miss**, non-GET, or `/api` URL → falls through to `OnReceivedRequest(request)`.

This method also runs from `OnDisconnected()` if a request's body was still pending when the client disconnected, so it can fire without headers ever fully completing in the usual receive path.

---

## Usage

```csharp
public class MySession : HttpSession
{
    public MySession(HttpServer server) : base(server) { }

    protected internal override void OnReceivedRequest(RequestModel request)
    {
        using var response = Lucifer.Rent<ResponseModel>();
        SendResponseAsync(response.MakeGetResponse("Hello"u8, "text/plain"u8));
    }
}
```

---

## Inherited API

All send, receive, disconnect, statistics, and base lifecycle hooks are inherited from `TcpSession`.

---