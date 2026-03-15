# UdsSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDS`

Unix Domain Socket (UDS) server-side session. Extends `StreamSessionTransport` — inherits batched async send and `SocketAsyncEventArgs`-driven receive. TCP socket options are not applied (`ApplySocketOptions()` is a no-op).

```csharp
public class UdsSession : StreamSessionTransport
```

---

## Constructor

```csharp
public UdsSession(UdsServer server)
```

Not instantiated directly — returned by `UdsServer.CreateSession()`.

---

## Inherited API

All send, receive, disconnect, statistics, and lifecycle hook methods are inherited from `StreamSessionTransport` and `SessionTransport`. See [StreamSessionTransport](transport-stream-session.md) and [SessionTransport](transport-session.md).
