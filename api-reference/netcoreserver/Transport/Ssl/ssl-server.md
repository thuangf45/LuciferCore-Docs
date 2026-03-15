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

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Context` | `SslContext` | The SSL context used for all sessions on this server |

---

## Session Factory

`SslServer` overrides `CreateSession()` to return `SslSession`:

```csharp
protected override SslSession CreateSession() => new(this);
```

Override this in your subclass to return your custom session type:

```csharp
[Server("ChatServer", 8443)]
public class ChatServer : SslServer
{
    public ChatServer(SslContext context, IPAddress address, int port)
        : base(context, address, port) { }

    protected override ChatSession CreateSession() => new(this);
}
```

---

## Inherited API

All server lifecycle, socket options, session management, and multicast methods are inherited from `ServerTransport`. See [ServerTransport](transport-server.md).

---

## Remarks

- The `Context` is shared across all sessions. Do not mutate it after the server has started.
- In `DEBUG` builds, use `SslContext.CreateDevelopmentContext()` to avoid needing a real certificate. See [SslContext](ssl-context.md).
