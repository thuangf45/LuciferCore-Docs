# StreamClientTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

`StreamClientTransport` is the default TCP-style client transport.

It is a concrete implementation of `ClientTransport` for stream sockets.

```csharp
public class StreamClientTransport : ClientTransport
```

---

## Construction

```csharp
public StreamClientTransport(string host)
public StreamClientTransport(DnsEndPoint endpoint)
public StreamClientTransport(IPAddress address, int port)
public StreamClientTransport(string address, int port)
public StreamClientTransport(IPEndPoint endpoint)
public StreamClientTransport(EndPoint endpoint, string address, int port)
```

Choose the constructor based on your input type (host, IP, or endpoint).

---

## Public surface

`StreamClientTransport` does not add extra public send/receive members beyond construction.

Use inherited APIs from `ClientTransport` (connect/disconnect, send/receive, dispose, and lifecycle behavior).

---

## Protected extensibility

For custom client behavior, derive from this type (or from `ClientTransport`) and override protected lifecycle hooks provided by the base transport.

---

## Notes

- Use this class as the default stream client transport.
- Keep protocol-specific logic in your derived client/session layer.