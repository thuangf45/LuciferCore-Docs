# TcpServer

**Namespace:** `LuciferCore.NetCoreServer.Transport.TCP`

`TcpServer` is the default TCP server class.  
It extends `ServerTransport` and creates one `TcpSession` per client.

```csharp
public class TcpServer : ServerTransport
```

---

## Constructors

```csharp
public TcpServer(IPAddress address, int port)
public TcpServer(string address, int port)
public TcpServer(IPEndPoint endpoint)
public TcpServer(DnsEndPoint endpoint)
```

Choose the constructor that matches your address type.

---

## Custom session type

Override `CreateSession()` in your server class:

```csharp
public class MyServer : TcpServer
{
    public MyServer(int port) : base(IPAddress.Any, port) { }

    protected override TcpSession CreateSession() => new MySession(this);
}
```

---

## Inherited API

`TcpServer` adds no new public server control methods.

Use inherited APIs from `ServerTransport`, including:

- `Start()`, `Stop()`, `Restart()`
- accept loop and session management
- `FindSession(...)`, `DisconnectAll()`
- `Multicast<T>(...)`
- lifecycle hooks/events
- `Dispose()`

---

## Notes

- Use `TcpServer` as the default TCP server transport.
- Add app logic by deriving custom server/session classes.