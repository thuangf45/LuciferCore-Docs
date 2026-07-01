# UdsServer

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDS`

`UdsServer` is the Unix Domain Socket server class.  
It extends `ServerTransport` and creates `UdsSession` for each client.

```csharp
public class UdsServer : ServerTransport
```

> UDS is supported on Linux and macOS.  
> On Windows, it requires Windows 10 build 17063 or later.

---

## Constructors

```csharp
public UdsServer(string path)                        // example: "/tmp/myapp.sock"
public UdsServer(UnixDomainSocketEndPoint endpoint)
```

Use a filesystem socket path instead of IP/port.

---

## Socket behavior

`UdsServer` uses Unix stream sockets (`AddressFamily.Unix`).

It overrides:

- `CreateSocket()` to create a Unix stream socket
- `ApplySocketOptions()` as empty (TCP-specific options are not used for UDS)

---

## Custom session type

Override `CreateSession()` in your server class:

```csharp
public class MyServer : UdsServer
{
    public MyServer(string path) : base(path) { }

    protected override UdsSession CreateSession() => new MySession(this);
}
```

---

## Inherited API

`UdsServer` adds no new public lifecycle methods.

Use inherited APIs from `ServerTransport`, including:

- `Start()`, `Stop()`, `Restart()`
- session management
- `FindSession(...)`, `DisconnectAll()`
- `Multicast<T>(...)`
- lifecycle hooks/events
- `Dispose()`

---

## Notes

- Use `UdsServer` for local IPC over Unix domain sockets.
- Keep app/protocol logic in derived server/session classes.