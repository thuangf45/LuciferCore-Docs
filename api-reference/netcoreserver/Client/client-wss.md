# WssClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

`WssClient` is the secure WebSocket client class (WebSocket over TLS).  
It extends `HttpsClient` and implements `IWebSocket`.

```csharp
public class WssClient : HttpsClient, IWebSocket
```

---

## Constructors

```csharp
public WssClient(SslContext context, string host)
public WssClient(SslContext context, DnsEndPoint endpoint)
public WssClient(SslContext context, IPAddress address, int port)
public WssClient(SslContext context, string address, int port)
public WssClient(SslContext context, IPEndPoint endpoint)
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

Connects (TLS + HTTP/WebSocket flow).

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
public class ChatClient : WssClient
{
    public ChatClient(SslContext context, string host, int port)
        : base(context, host, port) { }

    protected internal override void OnWsConnected(ResponseModel response)
        => Console.WriteLine("WSS connected");

    protected internal override void OnWsReceived(byte[] buffer, long offset, long size)
        => Console.WriteLine(Encoding.UTF8.GetString(buffer, (int)offset, (int)size));
}
```

---

## Inherited API

`WssClient` adds secure WebSocket client behavior.  
Other APIs are inherited from `HttpsClient` / `SslClient` / `ClientTransport`, including:

- TLS handshake/connect/disconnect
- HTTP request/response base flow
- send/receive base methods
- lifecycle hooks/events
- metrics/options access
- `Dispose()`
