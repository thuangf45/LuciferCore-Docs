# TcpSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.TCP`

Concrete TCP server-side session. Extends `StreamSessionTransport` — inherits double-buffered async send (main/flush buffer swap), `SocketAsyncEventArgs`-driven receive, and all `SessionTransport` lifecycle hooks.

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

All send, receive, disconnect, statistics, and lifecycle hook methods are inherited from `StreamSessionTransport` and `SessionTransport`.

---