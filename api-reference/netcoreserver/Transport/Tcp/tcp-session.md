# TcpSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.TCP`

Concrete TCP server-side session. Extends `StreamSessionTransport` — inherits batched async send, `SocketAsyncEventArgs`-driven receive, and all `SessionTransport` lifecycle hooks.

```csharp
public class TcpSession : StreamSessionTransport
```

---

## Constructor

```csharp
public TcpSession(TcpServer server)
```

Not instantiated directly — returned by `TcpServer.CreateSession()`.

---

## Inherited API

All send, receive, disconnect, statistics, and lifecycle hook methods are inherited from `StreamSessionTransport` and `SessionTransport`. See [StreamSessionTransport](transport-stream-session.md) and [SessionTransport](transport-session.md).
