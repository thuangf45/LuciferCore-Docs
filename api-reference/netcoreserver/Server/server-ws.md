# WsServer

**Namespace:** `LuciferCore.NetCoreServer.Server`

`WsServer` is the plain WebSocket server class (no TLS).  
It extends `HttpServer` and adds WebSocket multicast + close-all methods.

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

## Close all sessions

```csharp
bool CloseAll<T>(int status = 0, ReadOnlySpan<T> buffer = default) where T : unmanaged
```

Sends a WebSocket close frame to all sessions, then disconnects all.

---

## Custom session type

```csharp
protected override WsSession CreateSession() => new(this)
```

Example:

```csharp
public class MyWsServer : WsServer
{
    public MyWsServer(int port) : base(IPAddress.Any, port) { }

    protected override WsSession CreateSession() => new MyWsSession(this);
}
```

---

## Inherited API

`WsServer` adds WebSocket broadcast/close features.  
Other APIs are inherited from `HttpServer` / `ServerTransport`, including:

- `Start()`, `Stop()`, `Restart()`
- static content (`Cache`, `Mapping`, `AddStaticContent(...)`, ...)
- session management
- lifecycle hooks/events
- `Dispose()`