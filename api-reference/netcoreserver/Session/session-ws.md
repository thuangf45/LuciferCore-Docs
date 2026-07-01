# WsSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

`WsSession` is the plain WebSocket session class (no TLS).  
It extends `HttpSession` and implements `IWebSocket`.

```csharp
public class WsSession : HttpSession, IWebSocket
```

---

## Constructor

```csharp
public WsSession(WsServer server)
```

Created by `WsServer.CreateSession()`.  
You usually do not create it manually.

---

## Property

| Property | Type | Description |
|---|---|---|
| `IsWebSocket` | `bool` | `true` after HTTP→WebSocket upgrade is complete |

---

## Send API

```csharp
long SendText<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendTextAsync<T>(ReadOnlySpan<T> data) where T : unmanaged

long SendBinary<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendBinaryAsync<T>(ReadOnlySpan<T> data) where T : unmanaged

long SendClose<T>(int status, ReadOnlySpan<T> data) where T : unmanaged
bool SendCloseAsync<T>(int status, ReadOnlySpan<T> data) where T : unmanaged

long SendPing<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendPingAsync<T>(ReadOnlySpan<T> data) where T : unmanaged

long SendPong<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendPongAsync<T>(ReadOnlySpan<T> data) where T : unmanaged
```

Also available:

```csharp
public void SendUpgrade(ResponseModel response)
```

---

## Close API

```csharp
bool Close<T>(int status = 0, ReadOnlySpan<T> buffer = default) where T : unmanaged
```

Sends close frame, then disconnects.

---

## Main WebSocket hooks to override

| Method | When called |
|---|---|
| `OnWsConnecting(RequestModel, ResponseModel)` | Validate upgrade request |
| `OnWsConnected(...)` | Upgrade complete |
| `OnWsReceived(byte[], long, long)` | Frame data received (main handler) |
| `OnWsClose(...)` | Close frame received |
| `OnWsPing(...)` | Ping frame received |
| `OnWsPong(...)` | Pong frame received |
| `OnWsError(...)` | WebSocket error |

`OnWsReceived(...)` is the main method for WS message handling.

---

## Custom session example

```csharp
public class ChatSession : WsSession
{
    public ChatSession(WsServer server) : base(server) { }

    protected override void OnWsReceived(byte[] buffer, long offset, long size)
    {
        // handle incoming frame data
    }
}
```

---

## Inherited API

`WsSession` adds WebSocket behavior.  
Other APIs are inherited from `HttpSession` / `SessionTransport`, including:

- HTTP helpers (`SendResponse`, `Cache`, `Mapping`)
- send/receive base methods
- disconnect/dispose
- lifecycle hooks/events
- metrics/options access