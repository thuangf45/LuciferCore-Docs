# UdpServer

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDP`

UDP server. Extends `ServerTransport` for connectionless datagram communication. Unlike TCP/UDS, UDP has no persistent connections — the server receives datagrams from any remote endpoint and automatically creates or reuses a `UdpSession` per unique `EndPoint`.

```csharp
public class UdpServer : ServerTransport
```

---

## Constructors

```csharp
public UdpServer(IPAddress address, int port)
public UdpServer(string address, int port)
public UdpServer(IPEndPoint endpoint)
public UdpServer(DnsEndPoint endpoint)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Socket` | `Socket` | The underlying UDP socket (`SocketType.Dgram`, `ProtocolType.Udp`) |
| `DatagramsReceived` | `long` | Total datagrams received since last start |

---

## Socket Options

In addition to the standard options from `ServerTransport`, `UdpServer` supports:

| Property | Default | Description |
|---|---|---|
| `OptionReuseAddress` | `false` | Allow address reuse |
| `OptionExclusiveAddressUse` | `false` | Exclusive address use |
| `OptionDualMode` | `false` | IPv4+IPv6 dual mode (IPv6 sockets only) |

---

## Receive Architecture

UDP's connectionless nature requires a different receive architecture from TCP:

```
Socket.ReceiveFromAsync(_receiveEventArg)
    → ProcessReceiveFrom(e)
        → Lucifer.RentCopy(buffer)       ← copy datagram, zero-alloc pool
        → _receiveChannel.Writer.TryWrite()  ← bounded Channel (capacity 8192)
        → ReceiveAsync()                 ← immediately loop back
    ↓
ProcessReceiveLoop() (background Task)
    → _receiveChannel.Reader.WaitToReadAsync()
    → GetOrCreateSession(endpoint)       ← per-endpoint session registry
    → session.HandleReceived(buffer, offset, size)
    → Lucifer.Return(buffer)             ← return rented buffer to pool
```

Key design points:
- The socket receive loop and datagram dispatch run on **separate tasks** — the socket loop never blocks on session processing.
- The bounded `Channel` has capacity 8192 with `DropWrite` mode — datagrams are silently dropped if the channel is full rather than blocking the receive loop.
- Datagram buffers are rented from `Lucifer`'s pool and returned after dispatch. Never hold a reference to `buffer` outside `HandleReceived`.

---

## Session Management

Each unique remote `EndPoint` gets its own `UdpSession`, created lazily on first datagram receipt and cached in a `ConcurrentDictionary<EndPoint, UdpSession>`:

```csharp
// Auto-created on first datagram from a new endpoint
var session = GetOrCreateSession(remoteEndpoint);
```

Override `CreateSession()` to return a custom session type:

```csharp
public class MyUdpServer : UdpServer
{
    public MyUdpServer(int port) : base(IPAddress.Any, port) { }

    protected override UdpSession CreateSession() => new MyUdpSession(this);
}
```

---

## Start / Stop

`Start()` and `Stop()` are fully overridden for UDP:

- `Start()` — creates socket, binds, starts the background receive loop task, and calls `ReceiveAsync()`.
- `Stop()` — cancels the receive loop, completes the channel, closes and disposes the socket.

---

## Inherited API

Server lifecycle hooks (`OnStarted`, `OnStopped`, etc.) are inherited from `ServerTransport`. See [ServerTransport](transport-server.md).

---

## Remarks

- UDP is connectionless — `ConnectedSessions` reflects the number of unique endpoints seen, not persistent connections.
- There is no handshake or connection state per session. Sessions are created the moment the first datagram from a new endpoint arrives.
- `OptionReceiveBufferLimit` is respected — if the receive buffer needs to double beyond the limit, an error is sent and the datagram is dropped.
