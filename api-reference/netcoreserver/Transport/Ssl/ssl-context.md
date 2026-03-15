# SslContext

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

Configuration container for TLS/SSL connections. Holds the protocol version, certificate(s), and optional validation callback used by `SslServer`, `SslSession`, and `SslClient`.

```csharp
public class SslContext
```

---

## Constructors

```csharp
// TLS 1.3 default, no certificate
public SslContext()

// Specify protocol only
public SslContext(SslProtocols protocols)

// Protocol + custom validation callback (client-side, no server cert)
public SslContext(SslProtocols protocols, RemoteCertificateValidationCallback callback)

// Protocol + single certificate
public SslContext(SslProtocols protocols, X509Certificate certificate)

// Protocol + certificate + validation callback
public SslContext(SslProtocols protocols, X509Certificate certificate, RemoteCertificateValidationCallback callback)

// Protocol + certificate collection
public SslContext(SslProtocols protocols, X509Certificate2Collection certificates)

// Protocol + certificate collection + validation callback
public SslContext(SslProtocols protocols, X509Certificate2Collection certificates, RemoteCertificateValidationCallback callback)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Protocols` | `SslProtocols` | Enabled TLS/SSL protocol versions |
| `Certificate` | `X509Certificate` | Primary certificate for server authentication |
| `Certificates` | `X509Certificate2Collection` | Certificate collection (for SNI or multiple certs) |
| `CertificateValidationCallback` | `RemoteCertificateValidationCallback` | Custom certificate validation logic. `null` = default OS validation |
| `ClientCertificateRequired` | `bool` | When `true`, the server requests a client certificate during handshake. Connection is still accepted if the client does not provide one |

---

## `CreateDevelopmentContext()` *(static)*

```csharp
public static SslContext CreateDevelopmentContext()
```

Generates a self-signed RSA-2048 / SHA-256 certificate valid for `localhost` and `127.0.0.1`, then returns an `SslContext` with `SslProtocols.None` (OS-negotiated). The certificate is valid for 1 year from creation.

Used in `DEBUG` builds to avoid requiring a real certificate during development:

```csharp
public static SslContext CreateSslContext()
{
#if DEBUG
    return SslContext.CreateDevelopmentContext();
#else
    var cert = X509CertificateLoader.LoadPkcs12FromFile(s_certPath, s_certPassword);
    return new(SslProtocols.Tls12, cert);
#endif
}
```

The generated certificate includes:

- `SubjectAlternativeName` for `localhost` and `127.0.0.1`
- `KeyUsage`: `DigitalSignature` + `KeyEncipherment`
- `ExtendedKeyUsage`: `ServerAuthentication` (OID `1.3.6.1.5.5.7.3.1`)

---

## Usage

### Production server

```csharp
var cert = X509CertificateLoader.LoadPkcs12FromFile("server.pfx", "password");
var context = new SslContext(SslProtocols.Tls12, cert);
var server = new SslServer(context, IPAddress.Any, 8443);
```

### Client with custom validation (e.g. skip cert check in dev)

```csharp
var context = new SslContext(
    SslProtocols.Tls12,
    (sender, cert, chain, errors) => true // accept all — dev only
);
var client = new SslClient(context, "127.0.0.1", 8443);
```

### Mutual TLS (client certificate required)

```csharp
var context = new SslContext(SslProtocols.Tls13, serverCert)
{
    ClientCertificateRequired = true
};
```
