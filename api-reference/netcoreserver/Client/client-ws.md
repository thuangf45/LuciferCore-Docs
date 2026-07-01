# WsClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

Plain WebSocket client (unencrypted). Extends `HttpClient` and implements `IWebSocket` — performs the HTTP→WebSocket upgrade and provides the full WebSocket send/receive API.

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
Sends a close frame via **synchronous** `SendClose`, then calls `Disconnect()` (synchronous).

```csharp
public bool CloseAsync<T>(int status, ReadOnlySpan<T> buffer) where T : unmanaged
```
No default parameters. Notably still sends the close frame via **synchronous** `SendClose` (not `SendCloseAsync`), then calls `DisconnectAsync()`.

---

## Send API

All send methods are generic over `T : unmanaged`, so they accept either `ReadOnlySpan<byte>` or `ReadOnlySpan<char>` (routed through `WsClientProtocol.SendData` / `SendDataAsync`):

```csharp
// Text frames
long SendText<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendTextAsync<T>(ReadOnlySpan<T> data) where T : unmanaged

// Binary frames
long SendBinary<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendBinaryAsync<T>(ReadOnlySpan<T> data) where T : unmanaged

// Control frames
long SendClose<T>(int status, ReadOnlySpan<T> data) where T : unmanaged
bool SendCloseAsync<T>(int status, ReadOnlySpan<T> data) where T : unmanaged

long SendPing<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendPingAsync<T>(ReadOnlySpan<T> data) where T : unmanaged

long SendPong<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendPongAsync<T>(ReadOnlySpan<T> data) where T : unmanaged
```

---

## WebSocket Lifecycle Hooks

Override these `protected internal virtual` methods (all marked `AggressiveInlining`):

| Method | Default behavior | When called |
|---|---|---|
| `OnWsConnecting(RequestModel request)` | no-op | Fires from `OnConnected()`, before upgrade request handling |
| `OnWsConnecting(RequestModel request, ResponseModel response)` | returns `true` | Validate the upgrade before it's accepted. Return `false` to reject |
| `OnWsConnected(RequestModel request)` | no-op | Upgrade complete (request-based signature) |
| `OnWsConnected(ResponseModel response)` | no-op | Upgrade complete (response-based signature) |
| `OnWsDisconnecting()` | no-op | Before WebSocket disconnect |
| `OnWsDisconnected()` | no-op | After WebSocket disconnect |
| `OnWsReceived(byte[] buffer, long offset, long size)` | no-op | **Primary hook** — frame data received |
| `OnWsClose(byte[] buffer, long offset, long size, int status = 1000)` | `CloseAsync<byte>(status, [])` | Close frame received |
| `OnWsPing(byte[] buffer, long offset, long size)` | `SendPongAsync` with the ping payload | Ping frame received |
| `OnWsPong(byte[] buffer, long offset, long size)` | no-op | Pong frame received |
| `OnWsError(string error)` | `SendError(SocketError.SocketError)` | Protocol-level error |
| `OnWsError(SocketError error)` | `SendError(error)` | Socket-level error |

---

## Connection flow notes

`OnConnected()` is overridden to call `OnWsConnecting(Request)`. In the current code the lines that would actually send the upgrade request (`SendRequest(Request)` / `SendRequestAsync(Request)`) are **commented out** — worth being aware of when relying on this class as-is.

`IWebSocket.SendUpgrade(ResponseModel response)` is implemented as a **no-op** on `WsClient` (`{ }`), unlike on `WsSession`/`WssSession` where it forwards to `SendResponseAsync` — clients don't send upgrade *responses*, only requests.

---

## Usage

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

HTTP request sending and response hooks are inherited from `HttpClient`. Connect/disconnect methods are inherited from `ClientTransport`.

---