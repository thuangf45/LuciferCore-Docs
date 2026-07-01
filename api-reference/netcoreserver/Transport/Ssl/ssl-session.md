# SslSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

Server-side TLS/SSL session. Wraps the raw socket in an `SslStream` and performs the server-side TLS handshake asynchronously before enabling send/receive. All data is transparently encrypted/decrypted through the `SslStream`.

```csharp
public class SslSession : SessionTransport
```

---

## Constructor

```csharp
public SslSession(SslServer server)
```

Not instantiated directly — returned by `SslServer.CreateSession()`.

---

## Handshake State

| Property | Type | Description |
|---|---|---|
| `IsHandshaked` | `bool` (get only) | `true` after the TLS handshake completes successfully. `false` initially and after disconnect. |

> There is no public `IsHandshaking` property on this class. Handshake-in-progress state is not publicly exposed by `SslSession` itself.

Send and receive operations are silently rejected until `IsHandshaked` is `true`:

```csharp
public override bool SendAsync<T>(ReadOnlySpan<T> buffer)
{
    if (!IsHandshaked) return false;
    return base.SendAsync(buffer);
}

public override long Send<T>(ReadOnlySpan<T> buffer)
{
    if (!IsHandshaked) return 0;
    return base.Send(buffer);
}

public override long Receive(byte[] buffer, long offset, long size)
{
    if (!IsHandshaked) return 0;
    return base.Receive(buffer, offset, size);
}
```

---

## Handshake Flow

```csharp
Client connects
    → HandleHandshake()                         (internal)
        → SslStream.BeginAuthenticateAsServer()
        → OnHandshaking()                        ← override point
        → Server.OnHandshakingInternal(this)
    → ProcessHandshake() (async callback)         (internal)
        → SslStream.EndAuthenticateAsServer()
        → IsHandshaked = true
        → TryReceive()
        → OnHandshaked()                          ← override point
        → Server.OnHandshakedInternal(this)
```

The handshake uses the certificate, client-certificate-required flag, and protocols from the parent `SslServer.Context`. If the handshake throws, the session sends a socket error and disconnects.

Overriding `OnHandshaking()` and `OnHandshaked()` in a session subclass lets you hook into the TLS lifecycle without touching handshake internals. (Both are inherited hook methods; their exact accessibility is defined on the base class, not on `SslSession`.)

---

## Send / Receive

Public send/receive entry points all gate on `IsHandshaked` before delegating to the base implementation, which in turn drives data through the `SslStream` rather than the raw socket. The actual stream I/O (`_sslStream.Write`, `_sslStream.Read`, `_sslStream.BeginRead`, `_sslStream.BeginWrite`) is performed in internal overrides and is not part of the public surface.

---

## Inherited API

All other public send, receive, disconnect, statistics, and lifecycle hook methods are inherited from `SessionTransport`.

---

## Remarks

- `SslStream` is constructed with `leaveInnerStreamOpen: false` — the underlying `NetworkStream` is disposed when the SSL stream is disposed.
- If the parent `SslServer.Context` has a `CertificateValidationCallback`, it is passed into the `SslStream` constructor; otherwise the default validation is used.