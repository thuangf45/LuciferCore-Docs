# WsClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

`WsClient` is the plain WebSocket client class (no TLS).  
It extends `HttpClient` and implements `IWebSocket`.

```csharp
public class WsClient : HttpClient, IWebSocket
```

---

## Constructors

```csharp
public WsClient(string host)
public WsClient(DnsEndPoint endpoint)
public WsClient(IPAddress address, int port)
public WsClient(string address, int port)
public WsClient(IPEndPoint endpoint)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `IsWebSocket` | `bool` | `true` after WebSocket upgrade is complete |
| `WsNonce` | `byte[]` (protected) | Nonce used for `Sec-WebSocket-Key` |

---

## Connect / Close

```csharp
public override bool Connect()
```

Connects and starts WebSocket flow.

```csharp
public virtual bool Close<T>(int status = 0, ReadOnlySpan<T> buffer = default) where T : unmanaged
```

Sends close frame, then disconnects.

```csharp
public bool CloseAsync<T>(int status, ReadOnlySpan<T> buffer) where T : unmanaged
```

Async disconnect variant.

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

---

## Main WebSocket hooks to override

| Method | When called |
|---|---|
| `OnWsConnecting(...)` | Before upgrade |
| `OnWsConnected(...)` | Upgrade complete |
| `OnWsReceived(byte[], long, long)` | Frame data received (main handler) |
| `OnWsClose(...)` | Close frame received |
| `OnWsPing(...)` | Ping frame received |
| `OnWsPong(...)` | Pong frame received |
| `OnWsError(...)` | WebSocket error |

`OnWsReceived(...)` is the main method for WS message handling.

---

## Custom client example

```csharp
public class ChatClient : WsClient
{
    public ChatClient(string host, int port) : base(host, port) { }

    protected internal override void OnWsConnected(ResponseModel response)
        => Console.WriteLine("WebSocket connected");

    protected internal override void OnWsReceived(byte[] buffer, long offset, long size)
        => Console.WriteLine(Encoding.UTF8.GetString(buffer, (int)offset, (int)size));
}
```

---

## Inherited API

`WsClient` adds WebSocket client behavior.  
Other APIs are inherited from `HttpClient` / `ClientTransport`, including:

- HTTP request/response base flow
- connect/reconnect/disconnect
- send/receive base methods
- lifecycle hooks/events
- metrics/options access
- `Dispose()`