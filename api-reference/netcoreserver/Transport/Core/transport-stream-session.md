# StreamSessionTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

Concrete server-side stream session. Created by `ServerTransport` when a client connects. Implements async receive via `SocketAsyncEventArgs` and batched async send. Protocol-specific behavior (TLS handshake, WebSocket framing, etc.) is layered on top by higher-level session types that extend this class.

```csharp
public class StreamSessionTransport : SessionTransport
```

---

## Constructor

```csharp
public StreamSessionTransport(ServerTransport server)
```

Not instantiated directly — override `CreateSession()` in your server to return a subclass:

```csharp
protected override ChatSession CreateSession() => new(this);
```

---

## Send Pipeline

Outgoing sends are batched into a `List<ArraySegment<byte>>` and flushed in a single async syscall:

```
SendAsync(buffer)
    → HandleAsyncSend()  — adds ArraySegment to _batchList
    → HandleAsyncFlush() — Socket.SendAsync(_batchList) in one syscall
    → _batchList.Clear()
```

Multiple messages enqueued between flush cycles are sent as one kernel call, reducing syscall frequency significantly under load.

---

## Receive Pipeline

Driven by `SocketAsyncEventArgs` with a synchronous fast-path:

```
ReceiveAsync()
    → HandleTryReceive()      — sets buffer, calls Socket.ReceiveAsync
    → ProcessReceive(e)       — on completion: calls HandleProcessReceive(size)
    → OnReceived(buffer, ...) — your handler
    → loop
```

If `Socket.ReceiveAsync` completes synchronously, the loop continues immediately without re-entering the event system.

---

## Inherited API

All public send, receive, disconnect, and lifecycle hook methods are inherited from `SessionTransport`. See [SessionTransport](transport-session.md).

---

## Remarks

- `_batchList` is always accessed from within the single async send loop — no external synchronization needed.
- `HandleShutdown()` silently catches `SocketException` — the remote end may have already closed before `Shutdown` is called.
- All `internal virtual` hooks from `SessionTransport` are overridden — the internal API is sealed from application code.
