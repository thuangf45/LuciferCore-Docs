# TcpSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.TCP`

`TcpSession` is the default TCP server session class.  
It extends `StreamSessionTransport`.

```csharp
public class TcpSession : StreamSessionTransport
```

---

## Constructor

```csharp
public TcpSession(TcpServer server)
```

Created by `TcpServer.CreateSession()`.  
You usually do not create it manually.

---

## Inherited API

`TcpSession` adds no new public methods.

Use inherited APIs from `StreamSessionTransport` / `SessionTransport`, such as:

- `Send<T>(...)`
- `SendAsync<T>(...)`
- `Receive(...)`
- `Disconnect()`
- `Dispose()`
- lifecycle hooks/events

---

## Notes

- Use `TcpSession` for normal TCP server connections.
- For custom protocol logic, derive your own session class and override base hooks.