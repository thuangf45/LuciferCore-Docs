# WssClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

Secure WebSocket client (WSS). Extends `HttpsClient` and implements `IWebSocket` — performs TLS handshake, HTTP→WebSocket upgrade, and provides the full WebSocket send/receive API over an encrypted connection.

```csharp
public class WssClient : HttpsClient, IWebSocket
```

---

## Constructors

```csharp
public WssClient(SslContext context, IPAddress address, int port)
public WssClient(SslContext context, string address, int port)
public WssClient(SslContext context, IPEndPoint endpoint)
public WssClient(SslContext context, DnsEndPoint endpoint)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `IsWebSocket` | `bool` | `true` after TLS handshake + WebSocket upgrade both complete |
| `WsNonce` | `byte[]` | The 16-byte random nonce used for `Sec-WebSocket-Key` |

---

## Send API

Identical to `WsClient`. Client frames are **masked** per RFC 6455.

```csharp
bool SendTextAsync(ReadOnlySpan<byte> buffer)
bool SendBinaryAsync(ReadOnlySpan<byte> buffer)
bool SendCloseAsync(int status, ReadOnlySpan<byte> buffer)
bool SendPingAsync(ReadOnlySpan<byte> buffer)
bool SendPongAsync(ReadOnlySpan<byte> buffer)

// + sync variants: SendText, SendBinary, SendClose, SendPing, SendPong
```

## Close

```csharp
bool Close()                                          // sync Disconnect
bool CloseAsync()                                     // async DisconnectAsync
bool CloseAsync(int status, ReadOnlySpan<byte> buffer)
// + char overloads
```

## Synchronous Receive

```csharp
string ReceiveText()
Buffer ReceiveBinary()  // return to pool with Lucifer.Return() when done
```

---

## WebSocket Lifecycle Hooks

Same defaults as `WsClient`:

| Method | Default behavior | When called |
|---|---|---|
| `OnWsConnecting(RequestModel)` | — | Before upgrade request |
| `OnWsConnected(ResponseModel)` | — | Upgrade complete — `IsWebSocket` now `true` |
| `OnWsReceived(byte[] buffer, long offset, long size)` | — | **Primary hook** |
| `OnWsClose(...)` | `CloseAsync()` | Close frame received |
| `OnWsPing(...)` | `SendPongAsync(payload)` | Auto-pong |
| `OnWsDisconnecting()` / `OnWsDisconnected()` | — | Disconnect lifecycle |
| `OnWsError(string)` / `OnWsError(SocketError)` | Forward error | Protocol/socket error |

---

## Usage

```csharp
var context = new SslContext(SslProtocols.Tls12);
var client = new WssClient(context, "example.com", 443);
client.ConnectAsync();
client.SendBinaryAsync(packetBytes);

// Dev — skip cert validation
var devContext = new SslContext(SslProtocols.Tls12, (_, _, _, _) => true);
var devClient = new WssClient(devContext, "localhost", 8443);
```

---

## Inherited API

TLS state is inherited from `SslClient`. See [SslClient](ssl-client.md). HTTP request/response methods are inherited from `HttpsClient`. See [HttpsClient](client-https.md). Connect/disconnect is inherited from `ClientTransport`. See [ClientTransport](transport-client.md).
