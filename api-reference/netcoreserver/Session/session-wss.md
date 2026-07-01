# WssSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

Secure WebSocket session (WSS). Extends `HttpsSession` and implements `IWebSocket` — handles the HTTP→WebSocket upgrade and provides the full WebSocket send/receive API over an encrypted (TLS) connection.

```csharp
public class WssSession : HttpsSession, IWebSocket
```

---

## Constructor

```csharp
public WssSession(WssServer server)
```

Not instantiated directly — returned by `WssServer.CreateSession()`. Internally constructs the protocol handler: `_wsProtocol = new WsSessionProtocol(this, this)`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `IsWebSocket` | `bool` | `true` once `_wsProtocol.WsHandshaked` is `true` (i.e. after the WebSocket upgrade handshake completes, on top of the underlying TLS session) |

---

## Send API

All send methods are generic over `T : unmanaged`, so they accept either `ReadOnlySpan<byte>` or `ReadOnlySpan<char>` (routed through `WsSessionProtocol.SendData` / `SendDataAsync`) — identical shape to `WsSession`:

```csharp
// Text frames
long SendText<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendTextAsync<T>(ReadOnlySpan<T> data) where T : unmanaged

// Binary frames
long SendBinary<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendBinaryAsync<T>(ReadOnlySpan<T> data) where T : unmanaged

// Control frames
long SendClose<T>(int status, ReadOnlySpan<T> data) where T : unmanaged
bool SendCloseAsync<T>(int status, ReadOnlySpan<T> data) where T : unmanaged

long SendPing<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendPingAsync<T>(ReadOnlySpan<T> data) where T : unmanaged

long SendPong<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendPongAsync<T>(ReadOnlySpan<T> data) where T : unmanaged
```

Also available:

```csharp
public void SendUpgrade(ResponseModel response)
```

Sends the WebSocket upgrade HTTP response (`SendResponseAsync(response)`).

## Close

```csharp
bool Close<T>(int status = 0, ReadOnlySpan<T> buffer = default) where T : unmanaged
```

Single generic method with default parameters. Sends a close frame via `SendCloseAsync`, then calls `Disconnect()`.

---

## WebSocket Lifecycle Hooks

Override these `protected internal virtual` methods. They're also exposed through explicit `IWebSocket` interface implementations that forward to these same methods.

| Method | When called |
|---|---|
| `OnWsConnecting(RequestModel request)` | WebSocket connecting (no-op by default) |
| `OnWsConnecting(RequestModel request, ResponseModel response)` | Validate upgrade request before it's accepted. Return `false` to reject. Default returns `true` |
| `OnWsConnected(RequestModel request)` | Upgrade complete (request-based signature) |
| `OnWsConnected(ResponseModel response)` | Upgrade complete (response-based signature) |
| `OnWsDisconnecting()` | Before WebSocket disconnect |
| `OnWsDisconnected()` | After WebSocket disconnect |
| `OnWsReceived(byte[] buffer, long offset, long size)` | **Primary hook** — frame data received |
| `OnWsClose(byte[] buffer, long offset, long size, int status = 1000)` | Close frame received. Default calls `Close<byte>(status, [])` |
| `OnWsPing(byte[] buffer, long offset, long size)` | Ping frame received. Default replies with `SendPongAsync` |
| `OnWsPong(byte[] buffer, long offset, long size)` | Pong frame received (no-op by default) |
| `OnWsError(string error)` | Error occurred (string). Default calls `SendError(SocketError.SocketError)` |
| `OnWsError(SocketError error)` | Error occurred (SocketError). Default calls `SendError(error)` |

---

## Internal dispatch

`WssSession` overrides several `HttpsSession` hooks to route data through the WebSocket protocol once handshaked (`_wsProtocol.WsHandshaked`) — same pattern as `WsSession`:

- `OnReceived(byte[], long, long)` — dispatches to `_wsProtocol.OnReceived` when handshaked, otherwise `base.OnReceived`.
- `OnReceivedRequestHeader(RequestModel)` — if an `Upgrade` header is present and not yet handshaked, calls `_wsProtocol.PerformServerUpgrade(request)`.
- `OnReceivedRequest(RequestModel)` — when handshaked, forwards `request.BodySpan` to `_wsProtocol.OnReceived`; otherwise falls back to `base.OnReceivedRequest`.
- `OnReceivedRequestError(RequestModel, string)` — when handshaked, calls `SendError(new())` instead of base HTTP error handling.
- `OnReceivedRequestInternal(RequestModel)` — skipped entirely once `IsWebSocket` is `true`.
- `OnDisconnecting()` / `OnDisconnected()` — call `_wsProtocol.OnDisconnecting()` / `_wsProtocol.OnDisconnected()` for protocol cleanup, in addition to base cleanup.

---

## Usage

```csharp
public class ChatSession : WssSession
{
    public ChatSession(WssServer server) : base(server) { }

    protected override void OnWsReceived(byte[] buffer, long offset, long size)
    {
        // handle incoming frame data over TLS
    }
}
```

---

## Inherited API

TLS session state is inherited from `HttpsSession` / `SslSession`. HTTP response sending (`SendResponse`, `Cache`, `Mapping`) is inherited from `HttpsSession`. All base session methods are inherited from `SessionTransport`.

---