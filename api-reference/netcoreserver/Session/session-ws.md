# WsSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

Plain WebSocket session (unencrypted). Extends `HttpSession` and implements `IWebSocket` — handles the HTTP→WebSocket upgrade and provides the full WebSocket send/receive API.

```csharp
public class WsSession : HttpSession, IWebSocket
```

---

## Constructor

```csharp
public WsSession(WsServer server)
```

Not instantiated directly — returned by `WsServer.CreateSession()`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `IsWebSocket` | `bool` | `true` after the WebSocket upgrade handshake completes |

---

## Send API

All overloads accept both `ReadOnlySpan<byte>` and `ReadOnlySpan<char>`:

```csharp
// Text frames
long SendText(ReadOnlySpan<byte> buffer)
long SendText(ReadOnlySpan<char> text)
bool SendTextAsync(ReadOnlySpan<byte> buffer)
bool SendTextAsync(ReadOnlySpan<char> text)

// Binary frames
long SendBinary(ReadOnlySpan<byte> buffer)
long SendBinary(ReadOnlySpan<char> text)
bool SendBinaryAsync(ReadOnlySpan<byte> buffer)
bool SendBinaryAsync(ReadOnlySpan<char> text)

// Control frames
long SendClose(int status, ReadOnlySpan<byte> buffer)
long SendClose(int status, ReadOnlySpan<char> text)
bool SendCloseAsync(int status, ReadOnlySpan<byte> buffer)
bool SendCloseAsync(int status, ReadOnlySpan<char> text)

long SendPing(ReadOnlySpan<byte> buffer)
bool SendPingAsync(ReadOnlySpan<byte> buffer)

long SendPong(ReadOnlySpan<byte> buffer)
bool SendPongAsync(ReadOnlySpan<byte> buffer)
```

Prefer `*Async` variants on hot paths — they enqueue into the batched async send channel.

## Close

```csharp
bool Close()
bool Close(int status)
bool Close(int status, ReadOnlySpan<char> text)
bool Close(int status, ReadOnlySpan<byte> buffer)
```

Sends a Close frame then calls `Disconnect()`.

---

## Synchronous Receive

For scenarios where you need to pull data synchronously:

```csharp
string ReceiveText()    // blocks until a complete text message is assembled
Buffer ReceiveBinary()  // blocks until a complete binary message is assembled
```

The returned `Buffer` is pooled — call `Lucifer.Return(buffer)` when done.

---

## WebSocket Lifecycle Hooks

Override these in your session subclass:

| Method | When called |
|---|---|
| `OnWsConnecting(RequestModel)` | Before sending upgrade response |
| `OnWsConnecting(RequestModel, ResponseModel)` | Validate upgrade request. Return `false` to reject |
| `OnWsConnected(RequestModel)` | Upgrade complete — `IsWebSocket` is now `true` |
| `OnWsDisconnecting()` | Before WebSocket disconnect |
| `OnWsDisconnected()` | After WebSocket disconnect |
| `OnWsReceived(byte[] buffer, long offset, long size)` | **Primary hook** — binary/text frame data received |
| `OnWsClose(byte[] buffer, long offset, long size, int status)` | Close frame received |
| `OnWsPing(byte[] buffer, long offset, long size)` | Ping frame received |
| `OnWsPong(byte[] buffer, long offset, long size)` | Pong frame received |

---

## Usage

```csharp
[RateLimiter(10, 1)]
public partial class ChatSession : WsSession
{
    public ChatSession(WsServer server) : base(server) { }

    protected override void OnWsReceived(byte[] buffer, long offset, long size)
        => Lucifer.Dispatch(this, buffer, offset, size);
}
```

---

## Inherited API

HTTP request handling (`SendResponse`, `OnReceivedRequest`, `Cache`, `Mapping`) is inherited from `HttpSession`. See [HttpSession](session-http.md). All base session methods are inherited from `SessionTransport`. See [SessionTransport](transport-session.md).
