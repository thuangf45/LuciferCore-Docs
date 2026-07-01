# UdsSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDS`

Unix Domain Socket (UDS) server-side session. Extends `StreamSessionTransport` — inherits double-buffered async send (main/flush buffer swap) and `SocketAsyncEventArgs`-driven receive. TCP socket options are not applied (`ApplySocketOptions()` is overridden as a no-op).

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

## Socket Options

```csharp
protected override void ApplySocketOptions() { } // no-op
```

---

## Inherited API

All send, receive, disconnect, statistics, and lifecycle hook methods are inherited from `StreamSessionTransport` and `SessionTransport`.

---