# UdpClient

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDP`

UDP client for sending and receiving datagrams. Extends `ClientTransport` with UDP-specific socket setup, datagram statistics, multicast support, and endpoint-aware receive hooks.

```csharp
public class UdpClient : ClientTransport
```

---

## Constructors

```csharp
public UdpClient(IPAddress address, int port)
public UdpClient(string address, int port)
public UdpClient(IPEndPoint endpoint)
public UdpClient(DnsEndPoint endpoint)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `DatagramsSent` | `long` | Total datagrams sent since connect |
| `DatagramsReceived` | `long` | Total datagrams received since connect |

---

## Socket Options

| Property | Default | Description |
|---|---|---|
| `OptionReuseAddress` | `false` | Allow address reuse |
| `OptionExclusiveAddressUse` | `false` | Exclusive address use |
| `OptionMulticast` | `false` | When `true`, binds to `Endpoint` directly for multicast group membership. When `false`, binds to `0.0.0.0:0` (ephemeral port) |

---

## Socket & Bind

Uses `SocketType.Dgram` / `ProtocolType.Udp`. Unlike TCP, UDP does not call `Socket.Connect()` — it binds to a local address and sends datagrams to `Endpoint` via `Socket.SendTo`:

- `OptionMulticast = false` → binds to `0.0.0.0:0` (OS assigns ephemeral port)
- `OptionMulticast = true` → binds directly to `Endpoint` for multicast

---

## Send Pipeline

Datagrams are batched and sent as separate `SendToAsync` calls per flush cycle (UDP has no stream merging):

```
SendAsync(buffer)
    → HandleAsyncSend() — adds ArraySegment to _udpBatchList
    → HandleAsyncFlush()
        → foreach segment: Socket.SendToAsync(segment, Endpoint, token)
        → DatagramsSent++
        → _udpBatchList.Clear()
```

---

## Receive Pipeline

Uses `Socket.ReceiveFromAsync` to receive datagrams from any remote endpoint:

```
ReceiveAsync()
    → Socket.ReceiveFromAsync(_receiveEventArg)
    → ProcessReceive(e)
        → OnReceived(remoteEndpoint, buffer, offset, size)
            → filters: if remoteEndpoint matches Endpoint → OnReceived(buffer, offset, size)
        → ReceiveAsync()  ← loop
```

Datagrams from endpoints other than the configured `Endpoint` are silently filtered in `OnReceived(EndPoint, ...)`. Override this method to handle responses from arbitrary endpoints (e.g. multicast).

---

## Lifecycle Hooks

In addition to the standard `SessionTransport` hooks, `UdpClient` adds:

| Method | When called |
|---|---|
| `OnReceived(EndPoint endpoint, byte[] buffer, long offset, long size)` | When any datagram arrives — includes the sender's endpoint. Default implementation filters to `Endpoint` only |
| `OnSent(EndPoint endpoint, long sent)` | After a datagram is sent synchronously |

---

## Usage

```csharp
var client = new UdpClient("127.0.0.1", 9000);
client.Connect();
client.SendAsync("ping"u8);
```

Extend to override hooks:

```csharp
public class MyUdpClient : UdpClient
{
    public MyUdpClient(string host, int port) : base(host, port) { }

    protected override void OnConnected()
        => SendAsync("hello"u8);

    // Receives only from the configured Endpoint by default
    protected override void OnReceived(byte[] buffer, long offset, long size)
        => Console.WriteLine(Encoding.UTF8.GetString(buffer, (int)offset, (int)size));

    // Override to handle datagrams from any source (e.g. multicast)
    protected override void OnReceived(EndPoint endpoint, byte[] buffer, long offset, long size)
        => base.OnReceived(endpoint, buffer, offset, size);
}
```

---

## Inherited API

All connect/disconnect, statistics, and base lifecycle hooks are inherited from `ClientTransport` and `SessionTransport`. See [ClientTransport](transport-client.md) and [SessionTransport](transport-session.md).

---

## Remarks

- `DatagramsSent` and `DatagramsReceived` are reset on each `Connect()` via `ResetStatistic()`.
- UDP has no connection state in the OS — `IsConnected` reflects the client's logical state only.
- Buffer auto-doubling applies to the receive buffer: if a datagram fills the buffer exactly, the buffer doubles (subject to `OptionReceiveBufferLimit`).
