# HttpsSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

Secure HTTPS session. Extends `SslSession` and implements `IHttp` — handles TLS handshake, incremental HTTP request parsing, static content serving, and response sending over an encrypted connection.

```csharp
public class HttpsSession : SslSession, IHttp
```

---

## Constructor

```csharp
public HttpsSession(HttpsServer server)
```

Not instantiated directly — returned by `HttpsServer.CreateSession()`.

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

Send operations are silently rejected until `IsHandshaked` is `true` (TLS handshake complete). This is inherited behavior from `SslSession`.

---

## Lifecycle Hooks

Override these in your session subclass:

| Method | When called |
|---|---|
| `OnReceivedRequestHeader(RequestModel)` | Request headers parsed — body may still be pending |
| `OnReceivedRequest(RequestModel)` | **Primary hook** — full request ready. Override this to handle the request |
| `OnReceivedRequestError(RequestModel, string)` | Parse error — session will be disconnected |
| `OnReceivedCachedRequest(RequestModel, byte[])` | Static file cache hit — response sent automatically |
| `GetStaticPath(RequestModel)` | Override to customize static file path resolution |

---

## Usage

```csharp
public class ChatSession : HttpsSession
{
    public ChatSession(HttpsServer server) : base(server) { }

    protected internal override void OnReceivedRequest(RequestModel request)
    {
        Lucifer.Dispatch(this, request);
    }
}
```

---

## Inherited API

TLS handshake state (`IsHandshaked`, `IsHandshaking`) is inherited from `SslSession`. See [SslSession](ssl-session.md). All base session methods are inherited from `SessionTransport`. See [SessionTransport](transport-session.md).
