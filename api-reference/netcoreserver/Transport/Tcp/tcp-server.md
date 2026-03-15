# TcpServer

**Namespace:** `LuciferCore.NetCoreServer.Transport.TCP`

Concrete TCP server. Extends `ServerTransport` and creates `TcpSession` instances for each connected client.

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

---

## Session Factory

Override `CreateSession()` in your subclass to use a custom session type:

```csharp
public class MyServer : TcpServer
{
    public MyServer(int port) : base(IPAddress.Any, port) { }

    protected override TcpSession CreateSession() => new MySession(this);
}
```

---

## Inherited API

All server lifecycle, socket options, session management, and multicast methods are inherited from `ServerTransport`. See [ServerTransport](transport-server.md).
