# WssServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

`WssServer` is the secure WebSocket server class (WebSocket over TLS).  
It extends `HttpsServer` and adds WebSocket multicast + close-all methods.

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

---

## WebSocket multicast API

```csharp
bool MulticastText<T>(ReadOnlySpan<T> text) where T : unmanaged
bool MulticastBinary<T>(ReadOnlySpan<T> data) where T : unmanaged
bool MulticastPing<T>(ReadOnlySpan<T> data) where T : unmanaged
```

- `MulticastText(...)` sends a text frame
- `MulticastBinary(...)` sends a binary frame
- `MulticastPing(...)` sends a ping frame

---

## Data multicast API

```csharp
public bool MulticastData<T>(ReadOnlySpan<T> data, byte opcode, int status = 0) where T : unmanaged
public bool MulticastData(ReadOnlySpan<byte> data, byte opcode, int status = 0)
```

Use this when you want custom WebSocket opcodes.

---

## Close all sessions

```csharp
bool CloseAll<T>(int status = 0, ReadOnlySpan<T> buffer = default) where T : unmanaged
```

Sends a WebSocket close frame to all sessions, then disconnects all.

---

## Custom session type

```csharp
protected override WssSession CreateSession() => new(this)
```

Example:

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

`WssServer` adds secure WebSocket broadcast/close features.  
Other APIs are inherited from `HttpsServer` / `ServerTransport`, including:

- TLS/SSL server config
- `Start()`, `Stop()`, `Restart()`
- static content (`Cache`, `Mapping`, `AddStaticContent(...)`, ...)
- session management
- lifecycle hooks/events
- `Dispose()`