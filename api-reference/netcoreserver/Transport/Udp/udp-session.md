# UdpSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDP`

`UdpSession` is the server-side UDP session abstraction.  
It extends `SessionTransport` and represents a logical remote endpoint for datagram handling.

```csharp
public class UdpSession : SessionTransport
```

---

## Constructor

```csharp
public UdpSession(UdpServer server)
```

Created by `UdpServer.CreateSession()`.  
You typically use a derived class and let the server create it.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `DatagramsSent` | `long` | Total datagrams sent by this session |
| `Endpoint` | `EndPoint` | Remote endpoint associated with this session |

---

## Public API

| Method | Description |
|---|---|
| `Disconnect()` | Disconnects this logical UDP session |
| `Send<T>(ReadOnlySpan<T> data)` | Sends datagram to this session endpoint (sync) |
| `SendAsync<T>(ReadOnlySpan<T> data)` | Sends datagram to this session endpoint (async) |

---

## Hooks you can override

From `SessionTransport` (commonly used in UDP session subclasses):

- `OnConnected()`
- `OnDisconnecting()`
- `OnDisconnected()`
- `OnReceived(byte[] buffer, long offset, long size)`
- `OnSent(long sent, long pending)`
- `OnEmpty()`
- `OnSocketError` event subscription

---

## Example custom session

```csharp
public class EchoUdpSession : UdpSession
{
    public EchoUdpSession(UdpServer server) : base(server) { }

    protected override void OnReceived(byte[] buffer, long offset, long size)
    {
        Send<byte>(buffer.AsSpan((int)offset, (int)size));
    }
}
```

---

## Notes

- UDP session here is **logical**, not a TCP-style connected channel.
- Endpoint identity is datagram-based (`IP:Port`) and may change over time.
- Use app-level identifiers if you need long-lived user identity.