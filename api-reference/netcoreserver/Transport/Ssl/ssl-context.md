# SslContext

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

`SslContext` stores TLS/SSL settings.  
It is used by `SslServer`, `SslSession`, and `SslClient`.

```csharp
public class SslContext
```

---

## Constructors

```csharp
public SslContext()
public SslContext(SslProtocols protocols)
public SslContext(SslProtocols protocols, RemoteCertificateValidationCallback callback)
public SslContext(SslProtocols protocols, X509Certificate certificate)
public SslContext(SslProtocols protocols, X509Certificate certificate, RemoteCertificateValidationCallback callback)
public SslContext(SslProtocols protocols, X509Certificate2Collection certificates)
public SslContext(SslProtocols protocols, X509Certificate2Collection certificates, RemoteCertificateValidationCallback callback)
```

Use the constructor that matches your need: protocol, cert, cert collection, or validation callback.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Protocols` | `SslProtocols` | TLS/SSL protocol versions |
| `Certificate` | `X509Certificate` | Main server certificate |
| `Certificates` | `X509Certificate2Collection` | Certificate list |
| `CertificateValidationCallback` | `RemoteCertificateValidationCallback` | Custom cert validation callback |
| `ClientCertificateRequired` | `bool` | Ask client for certificate during handshake |

---

## Development helper

```csharp
public static SslContext CreateDevelopmentContext()
```

Creates a self-signed development certificate for local testing.

Good for local debug/dev runs.  
For production, use a real certificate.

---

## Quick usage

### Server

```csharp
var cert = X509CertificateLoader.LoadPkcs12FromFile("server.pfx", "password");
var context = new SslContext(SslProtocols.Tls12, cert);
var server = new SslServer(context, IPAddress.Any, 8443);
```

### Client with custom validation (dev only)

```csharp
var context = new SslContext(
    SslProtocols.Tls12,
    (sender, cert, chain, errors) => true
);
var client = new SslClient(context, "127.0.0.1", 8443);
```

### Mutual TLS

```csharp
var context = new SslContext(SslProtocols.Tls13, serverCert)
{
    ClientCertificateRequired = true
};
```

---

## Notes

- Use strict cert validation in production.
- `CreateDevelopmentContext()` is for local development/testing.