# SslSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

Server-side TLS/SSL session. Wraps the raw socket in an `SslStream` and performs the server-side TLS handshake asynchronously before enabling send/receive. All data is transparently encrypted through the `SslStream`.

```csharp
public class SslSession : SessionTransport
```

---

## Constructor

```csharp
public SslSession(SslServer server)
```

Not instantiated directly — returned by `SslServer.CreateSession()`.

---

## Handshake State

| Property | Type | Description |
|---|---|---|
| `IsHandshaking` | `bool` | `true` while the TLS handshake is in progress |
| `IsHandshaked` | `bool` | `true` after the TLS handshake completes successfully |

Send and receive operations are silently rejected (`return 0` / `return false`) until `IsHandshaked` is `true`.

---

## Handshake Flow

```
Client connects
    → HandleHandshake()
        → SslStream.BeginAuthenticateAsServer()
        → OnHandshaking()
    → ProcessHandshake() (async callback)
        → SslStream.EndAuthenticateAsServer()
        → IsHandshaked = true
        → ReceiveAsync()
        → OnHandshaked()
```

The handshake uses the certificate and protocol from the parent `SslServer.Context`. If the handshake fails, the session disconnects automatically.

---

## Send / Receive

All public `Send` and `SendAsync` overloads check `IsHandshaked` before proceeding. Data is written to and read from `SslStream` rather than the raw socket:

```csharp
// Send path: writes to SslStream
HandleSend()    → _sslStream.Write(buffer)
HandleAsyncSend() → _sslStream.WriteAsync(buffer, token)
HandleAsyncFlush() → _sslStream.FlushAsync(token)

// Receive path: reads from SslStream
HandleReceive() → _sslStream.Read(buffer, offset, size)
HandleTryReceive() → _sslStream.BeginRead(...)
```

Unlike `StreamSessionTransport`, SSL async sends are **not batched** — each `HandleAsyncSend` writes directly to `SslStream.WriteAsync`. Batching is handled at the TLS record layer by the OS.

---

## Inherited API

All public send, receive, disconnect, statistics, and lifecycle hook methods are inherited from `SessionTransport`. See [SessionTransport](transport-session.md).

---

## Remarks

- `SslStream` is created with `leaveInnerStreamOpen: false` — the underlying `NetworkStream` is disposed when the SSL stream is disposed.
- A `Guid` (`_sslStreamId`) is used to guard against stale async callbacks arriving after reconnection.
- `HandleShutdown()` disposes the `SslStream` first, then calls `Socket.Shutdown(SocketShutdown.Both)`.
- Overriding `OnHandshaking()` and `OnHandshaked()` in your session subclass lets you hook into the TLS lifecycle without touching handshake internals.
