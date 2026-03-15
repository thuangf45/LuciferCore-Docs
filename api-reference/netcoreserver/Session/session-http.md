# HttpSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

Plain HTTP session. Extends `TcpSession` and implements `IHttp` — handles incremental HTTP request parsing, static content serving, and response sending.

```csharp
public class HttpSession : TcpSession, IHttp
```

---

## Constructor

```csharp
public HttpSession(HttpServer server)
```

Not instantiated directly — returned by `HttpServer.CreateSession()`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Cache` | `FileCache` | Reference to the server's static content cache |
| `Mapping` | `Utf8Map<ByteString>` | Reference to the server's URL→file mapping |

---

## Send Response API

```csharp
long SendResponse(ResponseModel response)
long SendResponse(ReadOnlySpan<char> response)
long SendResponse(ReadOnlySpan<byte> response)

bool SendResponseAsync(ResponseModel response)
bool SendResponseAsync(ReadOnlySpan<char> response)
bool SendResponseAsync(ReadOnlySpan<byte> response)
```

Use `SendResponseAsync` on hot paths — it enqueues into the async send channel and returns immediately.

---

## Lifecycle Hooks

Override these in your session subclass:

| Method | When called |
|---|---|
| `OnReceivedRequestHeader(RequestModel)` | Request headers parsed — body may still be pending |
| `OnReceivedRequest(RequestModel)` | **Primary hook** — full request ready. Override this to handle the request |
| `OnReceivedRequestError(RequestModel, string)` | Parse error — session will be disconnected |
| `OnReceivedCachedRequest(RequestModel, byte[])` | A static file cache hit — response sent automatically. Override to intercept |
| `GetStaticPath(RequestModel)` | Override to customize static file path resolution |

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

In LuciferCore, `OnReceivedRequest` is wired to `Lucifer.Dispatch(this, request)` — the handler routing pipeline takes over from there.

---

## Inherited API

All send, receive, disconnect, statistics, and base lifecycle hooks are inherited from `TcpSession` and `SessionTransport`. See [SessionTransport](transport-session.md).
