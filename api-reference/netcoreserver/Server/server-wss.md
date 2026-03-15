# WssServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

Secure WebSocket server (WSS / WebSocket over TLS). Extends `HttpsServer` and implements `IWebSocket` — handles the TLS handshake, HTTP→WebSocket upgrade, and provides WebSocket multicast and session close APIs.

```csharp
public class WssServer : HttpsServer, IWebSocket
```

This is the server class used in LuciferCore's `[Server]` attribute examples — the primary server type for production deployments.

---

## Constructors

```csharp
public WssServer(SslContext context, IPAddress address, int port)
public WssServer(SslContext context, string address, int port)
public WssServer(SslContext context, IPEndPoint endpoint)
public WssServer(SslContext context, DnsEndPoint endpoint)
```

---

## WebSocket Multicast

Send a WebSocket frame to all connected sessions in a single pooled allocation:

```csharp
bool MulticastText(ReadOnlySpan<char> text)
bool MulticastText(ReadOnlySpan<byte> buffer)

bool MulticastBinary(ReadOnlySpan<char> text)
bool MulticastBinary(ReadOnlySpan<byte> buffer)

bool MulticastPing(ReadOnlySpan<char> text)
bool MulticastPing(ReadOnlySpan<byte> buffer)
```

---

## Close All Sessions

```csharp
bool CloseAll()
bool CloseAll(int status)
bool CloseAll(int status, ReadOnlySpan<char> text)
bool CloseAll(int status, ReadOnlySpan<byte> buffer)
```

Sends a WebSocket Close frame to all sessions and then calls `DisconnectAll()`.

---

## Session Factory

```csharp
protected override WssSession CreateSession()
```

Override to return a custom session type:

```csharp
[Server("ChatServer", 8443)]
public class ChatServer : WssServer
{
    public ChatServer(SslContext context, IPAddress address, int port)
        : base(context, address, port)
    {
        AddStaticContent(_staticContentPath);
        Cache.Freeze();

        Mapping = new(true)
        {
            { "/", "/index.html" },
            { "/404", "/404.html" }
        };
        Mapping.Freeze();
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public ChatServer(int port) : this(CreateSslContext(), IPAddress.Any, port) { }

    protected override ChatSession CreateSession() => new(this);
}
```

---

## IWebSocket Hooks

`WssServer` implements `IWebSocket` — override in your subclass to handle WebSocket lifecycle events at the server level:

| Method | When called |
|---|---|
| `OnWsConnecting(RequestModel, ResponseModel)` | Validate the upgrade request. Return `false` to reject |
| `OnWsConnected(RequestModel)` | Session upgrade complete |
| `OnWsDisconnecting()` / `OnWsDisconnected()` | Session closing / closed |

See [WebSocket Protocol Layer](ws-protocol.md) for the full `IWebSocket` interface.

---

## Inherited API

SSL configuration is inherited from `SslServer`. See [SslServer](ssl-server.md). Static content (`Cache`, `Mapping`, `AddStaticContent`) is inherited from `HttpsServer`. See [HttpsServer](server-https.md). All server lifecycle and session management methods are inherited from `ServerTransport`. See [ServerTransport](transport-server.md).

---

## Remarks

- `WssServer` serves both WSS and HTTPS on the same port — HTTP requests and WebSocket upgrades are dispatched through the same session.
- Use `SslContext.CreateDevelopmentContext()` in `DEBUG` builds. See [SslContext](ssl-context.md).
