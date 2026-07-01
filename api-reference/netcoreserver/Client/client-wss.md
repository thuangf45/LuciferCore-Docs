# WssClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

Secure WebSocket client (WSS). Extends `HttpsClient` and implements `IWebSocket` — performs the HTTP→WebSocket upgrade and provides the full WebSocket send/receive API over an encrypted (TLS) connection. Structurally identical to `WsClient`, differing only in base class and the `SslContext` parameter on every constructor.

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
| `IsWebSocket` | `bool` | `true` once `_wsProtocol.WsHandshaked` is `true` |
| `WsNonce` | `byte[]` (protected) | The nonce used for `Sec-WebSocket-Key`, exposed to subclasses via `_wsProtocol.WsNonce` |

Internal field `_syncConnect` tracks whether `Connect()` (sync) vs. an async connect path was used.

---

## Connect / Close

```csharp
public override bool Connect()
```
Sets `_syncConnect = true`, then calls `base.Connect()`.

```csharp
public virtual bool Close<T>(int status = 0, ReadOnlySpan<T> buffer = default) where T : unmanaged
```
Sends a close frame via **synchronous** `SendClose`, then calls `Disconnect()`.

```csharp
public bool CloseAsync<T>(int status, ReadOnlySpan<T> buffer) where T : unmanaged
```
No default parameters. Still sends the close frame via **synchronous** `SendClose` (not `SendCloseAsync`), then calls `DisconnectAsync()`.

---

## Send API

All send methods are generic over `T : unmanaged`, accepting either `ReadOnlySpan<byte>` or `ReadOnlySpan<char>` (routed through `WsClientProtocol.SendData` / `SendDataAsync`):

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

## WebSocket Lifecycle Hooks

| Method | Default behavior | When called |
|---|---|---|
| `OnWsConnecting(RequestModel request)` | no-op | Fires from `OnConnected()` |
| `OnWsConnecting(RequestModel request, ResponseModel response)` | returns `true` | Validate upgrade. Return `false` to reject |
| `OnWsConnected(RequestModel request)` | no-op | Upgrade complete (request-based signature) |
| `OnWsConnected(ResponseModel response)` | no-op | Upgrade complete (response-based signature) |
| `OnWsDisconnecting()` | no-op | Before disconnect |
| `OnWsDisconnected()` | no-op | After disconnect |
| `OnWsReceived(byte[] buffer, long offset, long size)` | no-op | **Primary hook** |
| `OnWsClose(byte[] buffer, long offset, long size, int status = 1000)` | `CloseAsync<byte>(status, [])` | Close frame received |
| `OnWsPing(byte[] buffer, long offset, long size)` | `SendPongAsync` with ping payload | Ping frame received |
| `OnWsPong(byte[] buffer, long offset, long size)` | no-op | Pong frame received |
| `OnWsError(string error)` | `SendError(SocketError.SocketError)` | Protocol-level error |
| `OnWsError(SocketError error)` | `SendError(error)` | Socket-level error |

---

## Connection flow notes

`OnConnected()` calls `OnWsConnecting(Request)`. As with `WsClient`, the lines that would send the upgrade request (`SendRequest(Request)` / `SendRequestAsync(Request)`) are **commented out** in the current source.

`IWebSocket.SendUpgrade(ResponseModel response)` is a **no-op** on `WssClient`, same as `WsClient` — clients don't send upgrade responses.

---

## Usage

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

TLS state is inherited from `HttpsClient` / `SslClient`. HTTP request/response methods are inherited from `HttpsClient`. Connect/disconnect is inherited from `ClientTransport`.

---