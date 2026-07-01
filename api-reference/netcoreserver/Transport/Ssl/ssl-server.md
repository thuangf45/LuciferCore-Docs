# SslServer

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

`SslServer` is the TLS/SSL server class.  
It extends `ServerTransport` and creates `SslSession` for each client.

```csharp
public class SslServer : ServerTransport
```

---

## Constructors

```csharp
public SslServer(SslContext context, IPAddress address, int port)
public SslServer(SslContext context, string address, int port)
public SslServer(SslContext context, IPEndPoint endpoint)
public SslServer(SslContext context, DnsEndPoint endpoint)
```

Each constructor sets server endpoint + SSL context.

---

## Property

| Property | Type | Description |
|---|---|---|
| `Context` | `SslContext` | SSL context used by all sessions |

---

## Custom session type

`SslServer` creates `SslSession` by default.

Override `CreateSession()` to use your own session type:

```csharp
public class ChatServer : SslServer
{
    public ChatServer(SslContext context, IPAddress address, int port)
        : base(context, address, port) { }

    protected override ChatSession CreateSession() => new(this);
}
```

---

## Inherited API

`SslServer` adds SSL context support.  
Other APIs are inherited from `ServerTransport`, including:

- `Start()`, `Stop()`, `Restart()`
- session management
- `FindSession(...)`, `DisconnectAll()`
- `Multicast<T>(...)`
- lifecycle hooks/events
- `Dispose()`