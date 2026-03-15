# SslClient

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

TLS/SSL client. Extends `ClientTransport` with an `SslContext` and performs a client-side TLS handshake after connecting. All data is transparently encrypted through `SslStream`.

```csharp
public class SslClient : ClientTransport
```

---

## Constructors

```csharp
public SslClient(SslContext context, IPAddress address, int port)
public SslClient(SslContext context, string address, int port)
public SslClient(SslContext context, IPEndPoint endpoint)
public SslClient(SslContext context, DnsEndPoint endpoint)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Context` | `SslContext` | The SSL context for this client connection |
| `IsHandshaking` | `bool` | `true` while the TLS handshake is in progress |
| `IsHandshaked` | `bool` | `true` after the TLS handshake completes successfully |

---

## Handshake Flow

```
Connect()
    → Socket connects
    → SslStream.BeginAuthenticateAsClient()
    → OnHandshaking()
    → ProcessHandshake() (async callback)
        → SslStream.EndAuthenticateAsClient()
        → IsHandshaked = true
        → ReceiveAsync()
        → OnHandshaked()
```

Uses the server name from `Address` for SNI (Server Name Indication). The certificate and protocol version come from `Context`.

---

## Send / Receive

`Send`, `SendAsync`, and `Receive` check `IsHandshaked` before proceeding — calls before the handshake completes silently return `0` or `false`.

All data is written to and read from `SslStream` rather than the raw socket.

---

## Usage

```csharp
var context = new SslContext(SslProtocols.Tls12);
var client = new SslClient(context, "example.com", 443);
client.Connect();

// Or extend and override hooks
public class MyClient : SslClient
{
    public MyClient() : base(new SslContext(), "example.com", 443) { }

    protected override void OnHandshaked()
        => SendAsync("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n");

    protected override void OnReceived(byte[] buffer, long offset, long size)
        => Console.WriteLine(Encoding.UTF8.GetString(buffer, (int)offset, (int)size));
}
```

### Skip certificate validation (dev only)

```csharp
var context = new SslContext(
    SslProtocols.Tls12,
    (sender, cert, chain, errors) => true
);
var client = new SslClient(context, "localhost", 8443);
```

---

## Inherited API

All connect/disconnect, send/receive, socket options, statistics, and lifecycle hook methods are inherited from `ClientTransport` and `SessionTransport`. See [ClientTransport](transport-client.md) and [SessionTransport](transport-session.md).

---

## Remarks

- A `Guid` is used to guard against stale async callbacks from a previous connection arriving after reconnect.
- `HandleShutdown()` disposes `SslStream` before calling `Socket.Shutdown` — same pattern as `SslSession`.
- `CreateSocket()` is overridden to create a `SocketType.Stream` / `ProtocolType.Tcp` socket, same as the stream transport base.
