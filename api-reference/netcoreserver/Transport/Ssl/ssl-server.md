# SslServer

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

TLS/SSL server. Extends `ServerTransport` with an `SslContext` and automatically creates `SslSession` instances for each connected client.

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

Each overload forwards `address`/`port`/`endpoint` to the matching `ServerTransport` base constructor, then assigns `Context`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Context` | `SslContext` | Get-only; the SSL context used for all sessions on this server, set once at construction |

---

## Session Factory

`SslServer` overrides `CreateSession()` to return `SslSession`:

```csharp
protected override SslSession CreateSession() => new(this);
```

Override this in your subclass to return a custom session type:

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

All server lifecycle, socket options, session management, and multicast methods are inherited from `ServerTransport`.

---