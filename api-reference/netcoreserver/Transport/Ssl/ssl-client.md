# SslClient

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

TLS/SSL client. Extends `ClientTransport` with an `SslContext` and performs a **synchronous** client-side TLS handshake right after the socket connects. All data is transparently encrypted/decrypted through `SslStream`.

```csharp
public class SslClient : ClientTransport
```

---

## Constructors

```csharp
public SslClient(SslContext context, string host)                          // default port 443
public SslClient(SslContext context, DnsEndPoint endpoint)
public SslClient(SslContext context, IPAddress address, int port)
public SslClient(SslContext context, string address, int port)
public SslClient(SslContext context, IPEndPoint endpoint)
public SslClient(SslContext context, EndPoint endpoint, string address, int port)  // main constructor
```

All overloads ultimately resolve to the `(SslContext, EndPoint, string, int)` constructor. `SslClient(context, host)` is a convenience overload that connects to `host` on port `443` via a `DnsEndPoint`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Context` | `SslContext` (get only) | The SSL context for this client connection |
| `IsHandshaking` | `bool` (get only) | `true` while the TLS handshake is in progress |
| `IsHandshaked` | `bool` (get only) | `true` after the TLS handshake completes successfully |

---

## Connect

```csharp
public override bool Connect()
```

Returns `false` immediately if a handshake is already in progress or already completed (`IsHandshaked || IsHandshaking`); otherwise delegates to the base TCP connect logic.

---

## Handshake Flow

Unlike the server-side session, the client performs the TLS handshake **synchronously** on the calling thread, not via `Begin/End` async pattern:

```
HandleHandshake()
    → new SslStream(...)
    → build SslClientAuthenticationOptions
         TargetHost = Address   (used for SNI)
         EnabledSslProtocols = Context.Protocols
         ClientCertificates = Context.Certificates / Context.Certificate
    → OnHandshaking()                     ← called BEFORE the handshake call
    → SslStream.AuthenticateAsClient(options)   ← blocking call, no async callback
    → IsHandshaked = true
    → OnHandshaked()                      ← called immediately after, on the same call
```

There is no `ProcessHandshake` callback and no automatic receive kicked off at the end of the handshake for the client (that pattern exists only on `SslSession`). If `AuthenticateAsClient` throws, the client sends a socket error and disconnects asynchronously, and the handshake method returns `false`.

---

## Send / Receive

`Send`, `SendAsync`, and `Receive` check **`IsHandshaked || IsHandshaking`** before proceeding — calls made before a handshake has even started return `0` or `false`. Note this is looser than `SslSession`: on the client, sends are allowed once the handshake has *started*, not only after it has fully completed.

```csharp
public override long Send<T>(ReadOnlySpan<T> buffer)
{
    if (!(IsHandshaked || IsHandshaking)) return 0;
    return base.Send(buffer);
}
```

All data is written to and read from `SslStream` rather than the raw socket; the actual stream I/O happens in internal overrides and is not part of the public surface.

---

## Usage

```csharp
var context = new SslContext(SslProtocols.Tls12);
var client = new SslClient(context, "example.com", 443);
client.Connect();
```

```csharp
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

> `OnHandshaking()`, `OnHandshaked()`, and `OnReceived(...)` are hook methods inherited from the base transport classes, not defined on `SslClient` itself — their exact accessibility/signature should be confirmed against `SessionTransport`/`ClientTransport`.

### Skip certificate validation (dev only)

```csharp
var context = new SslContext(
    SslProtocols.Tls12,
    (sender, cert, chain, errors) => true
);
var client = new SslClient(context, "localhost", 8443);
```

This relies on `SslContext` accepting a certificate validation callback — not verified from this file; confirm against `SslContext`'s own source.

---

## Inherited API

All connect/disconnect, send/receive, socket options, statistics, and lifecycle hook methods not listed above are inherited from `ClientTransport` and `SessionTransport`. See [ClientTransport](transport-client.md) and [SessionTransport](transport-session.md).

---

## Remarks

- `CreateSocket()` is overridden (`protected override Socket CreateSocket()`) to create a `SocketType.Stream` / `ProtocolType.Tcp` socket.
- An internal `Guid` (`_sslStreamId`) guards async receive/send callbacks against being processed after the stream has been replaced/torn down (relevant once data flow starts post-handshake).
- Disposal of the `SslStream` is handled by an internal method (`ShutdownSSLStream`), not a public `HandleShutdown()` — this is not part of the public API surface.
- Certificate revocation checking is explicitly disabled (`CertificateRevocationCheckMode = X509RevocationMode.NoCheck`) in the authentication options built by `HandleHandshake`.

---