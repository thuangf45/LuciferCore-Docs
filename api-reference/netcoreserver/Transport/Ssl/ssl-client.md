# SslClient

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

`SslClient` is the TLS/SSL client class.  
It extends `ClientTransport` and uses `SslStream` for encrypted I/O.

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
public SslClient(SslContext context, EndPoint endpoint, string address, int port)
```

Use the constructor that matches your input (host, IP, or endpoint).

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Context` | `SslContext` | SSL context used by this client |
| `IsHandshaking` | `bool` | `true` while handshake is running |
| `IsHandshaked` | `bool` | `true` after handshake is complete |

---

## Connect

```csharp
public override bool Connect()
```

Starts socket connect + TLS handshake.

If handshake is already running or already done, `Connect()` returns `false`.

---

## Handshake flow (simple)

1. connect socket
2. create `SslStream`
3. call `OnHandshaking()`
4. run TLS handshake
5. set `IsHandshaked = true`
6. call `OnHandshaked()`

If handshake fails, client reports error and disconnects.

---

## Send / Receive behavior

- Data goes through `SslStream` (encrypted)
- Send/receive calls are allowed when handshake has started/completed
- If handshake has not started, send/receive returns `false`/`0`

---

## Quick usage

```csharp
var context = new SslContext(SslProtocols.Tls12);
var client = new SslClient(context, "example.com", 443);
client.Connect();
```

---

## Custom client example

```csharp
public class MyClient : SslClient
{
    public MyClient() : base(new SslContext(), "example.com", 443) { }

    protected override void OnHandshaked()
        => SendAsync("GET / HTTP/1.1\r\nHost: example.com\r\n\r\n");

    protected override void OnReceived(byte[] buffer, long offset, long size)
        => Console.WriteLine(Encoding.UTF8.GetString(buffer, (int)offset, (int)size));
}
```

---

## Inherited API

`SslClient` adds TLS behavior.  
Other APIs are inherited from `ClientTransport` / `SessionTransport`, including:

- connect/reconnect/disconnect
- `Send<T>(...)`, `SendAsync<T>(...)`
- receive methods/hooks
- lifecycle hooks/events
- metrics/options access
- `Dispose()`

---

## Notes

- Default HTTPS port is `443` in `SslClient(context, host)`.
- `SslContext` controls protocols/certificates/validation behavior.