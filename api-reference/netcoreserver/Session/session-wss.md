# WssSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

`WssSession` is the secure WebSocket session class (WebSocket over TLS).  
It extends `HttpsSession` and implements `IWebSocket`.

```csharp
public class WssSession : HttpsSession, IWebSocket
```

---

## Constructor

```csharp
public WssSession(WssServer server)
```

Created by `WssServer.CreateSession()`.  
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
public class ChatSession : WssSession
{
    public ChatSession(WssServer server) : base(server) { }

    protected override void OnWsReceived(byte[] buffer, long offset, long size)
    {
        // handle incoming frame data over TLS
    }
}
```

---

## Inherited API

`WssSession` adds secure WebSocket behavior.  
Other APIs are inherited from `HttpsSession` / `SslSession` / `SessionTransport`, including:

- TLS session behavior
- HTTP helpers (`SendResponse`, `Cache`, `Mapping`)
- send/receive base methods
- disconnect/dispose
- lifecycle hooks/events
- metrics/options access