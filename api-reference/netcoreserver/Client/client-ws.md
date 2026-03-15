# WsClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

Plain WebSocket client (unencrypted). Extends `HttpClient` and implements `IWebSocket` — performs the HTTP→WebSocket upgrade and provides the full WebSocket send/receive API.

```csharp
public class WsClient : HttpClient, IWebSocket
```

---

## Constructors

```csharp
public WsClient(IPAddress address, int port)
public WsClient(string address, int port)
public WsClient(IPEndPoint endpoint)
public WsClient(DnsEndPoint endpoint)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `IsWebSocket` | `bool` | `true` after the WebSocket upgrade completes |
| `WsNonce` | `byte[]` | The 16-byte random nonce used for `Sec-WebSocket-Key`. Exposed for testing |

---

## Send API

All overloads accept both `ReadOnlySpan<byte>` and `ReadOnlySpan<char>`. Client frames are **masked** per RFC 6455.

```csharp
long SendText(ReadOnlySpan<byte> buffer)
bool SendTextAsync(ReadOnlySpan<byte> buffer)

long SendBinary(ReadOnlySpan<byte> buffer)
bool SendBinaryAsync(ReadOnlySpan<byte> buffer)

long SendClose(int status, ReadOnlySpan<byte> buffer)
bool SendCloseAsync(int status, ReadOnlySpan<byte> buffer)

long SendPing(ReadOnlySpan<byte> buffer)
bool SendPingAsync(ReadOnlySpan<byte> buffer)

long SendPong(ReadOnlySpan<byte> buffer)
bool SendPongAsync(ReadOnlySpan<byte> buffer)
```

## Close

```csharp
bool Close()
bool Close(int status)
bool Close(int status, ReadOnlySpan<char> text)
bool Close(int status, ReadOnlySpan<byte> buffer)  // sync Disconnect

bool CloseAsync()
bool CloseAsync(int status)
bool CloseAsync(int status, ReadOnlySpan<char> text)
bool CloseAsync(int status, ReadOnlySpan<byte> buffer)  // async DisconnectAsync
```

## Synchronous Receive

```csharp
string ReceiveText()
Buffer ReceiveBinary()  // return to pool with Lucifer.Return() when done
```

---

## WebSocket Lifecycle Hooks

Default behaviors are provided — override to customize:

| Method | Default behavior | When called |
|---|---|---|
| `OnWsConnecting(RequestModel)` | — | Before sending upgrade request |
| `OnWsConnected(ResponseModel)` | — | Upgrade response received |
| `OnWsReceived(byte[] buffer, long offset, long size)` | — | **Primary hook** — frame data received |
| `OnWsClose(...)` | `CloseAsync()` | Close frame received |
| `OnWsPing(...)` | `SendPongAsync(payload)` | Ping frame — auto-pong by default |
| `OnWsPong(...)` | — | Pong frame received |
| `OnWsDisconnecting()` / `OnWsDisconnected()` | — | Disconnect lifecycle |
| `OnWsError(string)` | `SendError(SocketError.SocketError)` | Protocol error |
| `OnWsError(SocketError)` | `SendError(error)` | Socket error |

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

var client = new ChatClient("example.com", 80);
client.ConnectAsync();
client.SendBinaryAsync(packetBytes);
```

---

## Inherited API

HTTP request sending and response hooks are inherited from `HttpClient`. See [HttpClient](client-http.md). Connect/disconnect methods are inherited from `ClientTransport`. See [ClientTransport](transport-client.md).
