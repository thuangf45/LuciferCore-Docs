# WssServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

Secure WebSocket server (WSS / WebSocket over TLS). Extends `HttpsServer` — handles WebSocket multicast and session close APIs over a TLS connection.

```csharp
public class WssServer : HttpsServer
```

---

## Constructors

```csharp
public WssServer(SslContext context, IPAddress address, int port)
public WssServer(SslContext context, string address, int port)
public WssServer(SslContext context, DnsEndPoint endpoint)
public WssServer(SslContext context, IPEndPoint endpoint)
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

The underlying multi-purpose entry point is exposed as `public` on `WssServer` (unlike `WsServer`, where the byte-span overload is `private`):

```csharp
public bool MulticastData<T>(ReadOnlySpan<T> data, byte opcode, int status = 0) where T : unmanaged
public bool MulticastData(ReadOnlySpan<byte> data, byte opcode, int status = 0)
```

`char` spans are appended into a pooled `Buffer` and sent as text bytes; `byte` spans are sent directly; other unmanaged types are reinterpreted as bytes via `MemoryMarshal.Cast`.

---

## Close All Sessions

```csharp
bool CloseAll<T>(int status = 0, ReadOnlySpan<T> buffer = default) where T : unmanaged
```

Single generic method with default parameters (not separate overloads). Sends a WebSocket Close frame (`WS_FIN | WS_CLOSE`) with the given status/payload to all sessions, then calls `DisconnectAll()`.

---

## Session Factory

```csharp
protected override WssSession CreateSession() => new(this)
```

Override to return a custom session type:

```csharp
public class MyWssServer : WssServer
{
    public MyWssServer(SslContext context, int port)
        : base(context, IPAddress.Any, port) { }

    protected override WssSession CreateSession() => new MyWssSession(this);
}
```

---

## Inherited API

SSL configuration and static content (`Cache`, `Mapping`, `AddStaticContent`) are inherited from `HttpsServer`. All server lifecycle and session management methods are inherited from `ServerTransport`.

---