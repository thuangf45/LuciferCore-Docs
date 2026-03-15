# SessionTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport`

Base class for all server-side sessions. Manages a single client connection: send/receive pipeline, async Channel-based send loop, statistics, and lifecycle hooks.

```csharp
public class SessionTransport : IDisposable
```

---

## Identity & Connection State

| Property | Type | Description |
|---|---|---|
| `Id` | `long` | Unique session identifier assigned by the server |
| `Server` | `ServerTransport` | The server that owns this session |
| `Socket` | `Socket` | The underlying socket for this connection |
| `IsConnected` | `bool` | `true` while the session has an active connection |
| `IsDisposed` | `bool` | `true` after `Dispose()` has been called |
| `IsSocketDisposed` | `bool` | `true` after the underlying socket has been released |

---

## Options

| Property | Default | Description |
|---|---|---|
| `OptionSendBufferSize` | `8192` | Initial send buffer size in bytes |
| `OptionSendBufferLimit` | `0` | Max send buffer size. `0` = unlimited |
| `OptionReceiveBufferSize` | `8192` | Initial receive buffer size in bytes |
| `OptionReceiveBufferLimit` | `0` | Max receive buffer size. `0` = unlimited |

---

## Statistics

| Property | Type | Description |
|---|---|---|
| `BytesPending` | `long` | Bytes waiting in the send queue |
| `BytesSending` | `long` | Bytes currently being sent |
| `BytesSent` | `long` | Total bytes sent since connection |
| `BytesReceived` | `long` | Total bytes received since connection |

---

## Send API

```csharp
long Send(ReadOnlySpan<byte> buffer)     // synchronous send
long Send(ReadOnlySpan<char> text)       // UTF-8 encode then send
bool SendAsync(ReadOnlySpan<byte> data)  // enqueue for async send
bool SendAsync(ReadOnlySpan<char> text)  // enqueue for async send
bool SendAsync(Buffer buffer)            // enqueue pooled Buffer
```

## Receive API

```csharp
long   Receive(byte[] buffer)
long   Receive(byte[] buffer, long offset, long size)
string Receive(long size)                // receive as UTF-8 string
void   ReceiveAsync()                    // start async receive loop
```

## Disconnect

```csharp
bool Disconnect()
```

---

## Lifecycle Hooks

Override these in your session subclass:

| Method | When called |
|---|---|
| `OnConnecting()` | Before connection is established |
| `OnConnected()` | After connection is fully established |
| `OnHandshaking()` | During TLS/WebSocket handshake |
| `OnHandshaked()` | After handshake completes |
| `OnDisconnecting()` | Before disconnect |
| `OnDisconnected()` | After disconnect |
| `OnReceived(byte[] buffer, long offset, long size)` | When data arrives — primary receive hook |
| `OnSent(long sent, long pending)` | After data is sent |
| `OnEmpty()` | When the send queue drains to zero |

---

## Events

```csharp
event Action<SocketError>? OnSocketError
```

Raised on socket errors. Subscribe to log or handle transport-level failures.
