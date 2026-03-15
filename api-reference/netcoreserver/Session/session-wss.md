# WssSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

Secure WebSocket session (WSS). Extends `HttpsSession` and implements `IWebSocket` — handles TLS handshake, HTTP→WebSocket upgrade, and provides the full WebSocket send/receive API over an encrypted connection.

```csharp
public class WssSession : HttpsSession, IWebSocket
```

This is the session class used in LuciferCore's `[Server]` attribute examples — the primary session type for production deployments.

---

## Constructor

```csharp
public WssSession(WssServer server)
```

Not instantiated directly — returned by `WssServer.CreateSession()`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `IsWebSocket` | `bool` | `true` after the WebSocket upgrade handshake completes (requires TLS handshake first) |

---

## Send API

Identical to `WsSession`. All overloads accept both `ReadOnlySpan<byte>` and `ReadOnlySpan<char>`:

```csharp
bool SendTextAsync(ReadOnlySpan<byte> buffer)
bool SendBinaryAsync(ReadOnlySpan<byte> buffer)
bool SendCloseAsync(int status, ReadOnlySpan<byte> buffer)
bool SendPingAsync(ReadOnlySpan<byte> buffer)
bool SendPongAsync(ReadOnlySpan<byte> buffer)

// + sync variants: SendText, SendBinary, SendClose, SendPing, SendPong
```

## Close

```csharp
bool Close()
bool Close(int status)
bool Close(int status, ReadOnlySpan<char> text)
bool Close(int status, ReadOnlySpan<byte> buffer)
```

## Synchronous Receive

```csharp
string ReceiveText()
Buffer ReceiveBinary()  // return Buffer to pool with Lucifer.Return() when done
```

---

## WebSocket Lifecycle Hooks

Override these in your session subclass:

| Method | When called |
|---|---|
| `OnWsConnecting(RequestModel)` | Before sending upgrade response |
| `OnWsConnecting(RequestModel, ResponseModel)` | Validate upgrade request. Return `false` to reject |
| `OnWsConnected(RequestModel)` | Upgrade complete — `IsWebSocket` is now `true` |
| `OnWsDisconnecting()` / `OnWsDisconnected()` | WebSocket disconnect lifecycle |
| `OnWsReceived(byte[] buffer, long offset, long size)` | **Primary hook** — frame data received |
| `OnWsClose(byte[] buffer, long offset, long size, int status)` | Close frame received |
| `OnWsPing(byte[] buffer, long offset, long size)` | Ping frame received |
| `OnWsPong(byte[] buffer, long offset, long size)` | Pong frame received |

---

## Usage

```csharp
[RateLimiter(10, 1)]
public partial class ChatSession : WssSession
{
    public ChatSession(WssServer server) : base(server) { }

    // Forward WebSocket binary frames to the dispatch pipeline
    protected override void OnWsReceived(byte[] buffer, long offset, long size)
        => Lucifer.Dispatch(this, buffer, offset, size);

    // Forward HTTP requests to the dispatch pipeline
    protected internal override void OnReceivedRequest(RequestModel request)
        => Lucifer.Dispatch(this, request);
}
```

---

## Inherited API

TLS state (`IsHandshaked`) is inherited from `SslSession`. See [SslSession](ssl-session.md). HTTP response sending (`SendResponse`, `Cache`, `Mapping`) is inherited from `HttpsSession`. See [HttpsSession](session-https.md). All base session methods are inherited from `SessionTransport`. See [SessionTransport](transport-session.md).
