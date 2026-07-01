# WsServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

Plain WebSocket server (unencrypted). Extends `HttpServer` — handles WebSocket multicast and session close APIs.

```csharp
public class WsServer : HttpServer
```

---

## Constructors

```csharp
public WsServer(IPAddress address, int port)
public WsServer(string address, int port)
public WsServer(DnsEndPoint endpoint)
public WsServer(IPEndPoint endpoint)
```

All constructors zero-fill the internal `WsSendMask` (4-byte mask used when building outgoing frames).

---

## WebSocket Multicast

Send a WebSocket frame to all connected sessions in a single pooled allocation. Each method is generic over `T : unmanaged`, so it accepts either `ReadOnlySpan<char>` or `ReadOnlySpan<byte>` (and other unmanaged types via a raw byte-cast fallback):

```csharp
bool MulticastText<T>(ReadOnlySpan<T> text) where T : unmanaged
bool MulticastBinary<T>(ReadOnlySpan<T> data) where T : unmanaged
bool MulticastPing<T>(ReadOnlySpan<T> data) where T : unmanaged
```

- `MulticastText` sends `WS_FIN | WS_TEXT`
- `MulticastBinary` sends `WS_FIN | WS_BINARY`
- `MulticastPing` sends `WS_FIN | WS_PING`

Internally, `char` spans are appended into a pooled `Buffer` and sent as UTF-8/text bytes; `byte` spans are sent directly; any other unmanaged type is reinterpreted as bytes via `MemoryMarshal.Cast`.

---

## Close All Sessions

```csharp
bool CloseAll<T>(int status = 0, ReadOnlySpan<T> buffer = default) where T : unmanaged
```

Single generic method with default parameters (not separate overloads). Sends a WebSocket Close frame (`WS_FIN | WS_CLOSE`) with the given status/payload to all sessions, then calls `DisconnectAll()`.

---

## Session Factory

```csharp
protected override WsSession CreateSession() => new(this)
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

## Inherited API

Static content (`Cache`, `Mapping`, `AddStaticContent`) and all server lifecycle / session management methods are inherited from `HttpServer` / `ServerTransport`.

---