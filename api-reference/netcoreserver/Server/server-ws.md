# WsServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

Plain WebSocket server (unencrypted). Extends `HttpServer` and implements `IWebSocket` — handles the HTTP→WebSocket upgrade and provides WebSocket multicast and session close APIs.

```csharp
public class WsServer : HttpServer, IWebSocket
```

---

## Constructors

```csharp
public WsServer(IPAddress address, int port)
public WsServer(string address, int port)
public WsServer(IPEndPoint endpoint)
public WsServer(DnsEndPoint endpoint)
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
protected override WsSession CreateSession()
```

Override to return a custom session type:

```csharp
public class MyWsServer : WsServer
{
    public MyWsServer(int port) : base(IPAddress.Any, port) { }

    protected override WsSession CreateSession() => new MyWsSession(this);
}
```

---

## IWebSocket Hooks

`WsServer` implements `IWebSocket` — override these in your subclass to handle WebSocket lifecycle events at the server level:

| Method | When called |
|---|---|
| `OnWsConnecting(RequestModel, ResponseModel)` | Validate the upgrade request. Return `false` to reject |
| `OnWsConnected(RequestModel)` | Session upgrade complete |
| `OnWsDisconnecting()` / `OnWsDisconnected()` | Session closing / closed |

See [WebSocket Protocol Layer](ws-protocol.md) for the full `IWebSocket` interface.

---

## Inherited API

Static content (`Cache`, `Mapping`, `AddStaticContent`) is inherited from `HttpServer`. See [HttpServer](server-http.md). All server lifecycle and session management methods are inherited from `ServerTransport`. See [ServerTransport](transport-server.md).
