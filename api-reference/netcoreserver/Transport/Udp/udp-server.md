# UdpServer

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDP`

`UdpServer` is the UDP server class.  
It extends `ServerTransport` and creates `UdpSession` for incoming datagrams.

```csharp
public class UdpServer : ServerTransport
```

---

## Constructors

```csharp
public UdpServer(IPAddress address, int port)
public UdpServer(string address, int port)
public UdpServer(DnsEndPoint endpoint)
public UdpServer(IPEndPoint endpoint)
```

Each constructor sets the UDP bind endpoint.

---

## Property

| Property | Type | Description |
|---|---|---|
| `Socket` | `Socket` | Underlying UDP socket used by the server |

---

## Public API

| Method | Description |
|---|---|
| `Start()` | Starts UDP server and begins receive loop |
| `Stop()` | Stops server and releases socket/resources |
| `ReceiveAsync()` | Triggers async receive loop |
| `Send<T>(EndPoint endpoint, ReadOnlySpan<T> data)` | Sends one datagram to a specific endpoint (sync) |
| `SendAsync<T>(EndPoint endpoint, ReadOnlySpan<T> data)` | Sends one datagram to a specific endpoint (async) |

---

## Protected hooks you can override

| Method | Description |
|---|---|
| `CreateSocket()` | Customize UDP socket creation |
| `CreateSession()` | Return your custom `UdpSession` type |
| `OnReceived(EndPoint endpoint, byte[] buffer, long offset, long size)` | Called when a datagram is received |
| `OnSent(EndPoint endpoint, long sent)` | Called when a datagram send completes |
| `OnError(SocketError error)` | Called on socket-level errors |

---

## Custom session type

`UdpServer` creates `UdpSession` by default.

Override `CreateSession()` to use your own session type:

```csharp
public class EchoUdpServer : UdpServer
{
    public EchoUdpServer(int port) : base(IPAddress.Any, port) { }

    protected override UdpSession CreateSession() => new EchoUdpSession(this);
}
```

---

## Inherited API

`UdpServer` adds UDP datagram behavior.  
Other APIs are inherited from `ServerTransport`, including:

- lifecycle (`Start`, `Stop`, `Restart`)
- session registry utilities
- server lifecycle hooks/events
- metrics/options access
- `Dispose()`