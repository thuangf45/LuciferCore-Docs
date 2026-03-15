# UdpSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDP`

Server-side UDP session bound to a specific remote `EndPoint`. Unlike TCP sessions, a `UdpSession` does not own its socket — it shares the server's single UDP socket and uses `Socket.SendTo` / `Socket.SendToAsync` to address outgoing datagrams to its specific endpoint.

```csharp
public class UdpSession : SessionTransport
```

---

## Constructor

```csharp
public UdpSession(UdpServer server)
```

Not instantiated directly — created by `UdpServer.GetOrCreateSession()` on first datagram from a new endpoint.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Endpoint` | `EndPoint?` | The remote endpoint this session represents |
| `DatagramsSent` | `long` | Total datagrams sent by this session |

---

## Send Pipeline

Sends use `Socket.SendTo` / `Socket.SendToAsync` targeting `Endpoint`. Outgoing datagrams are batched per flush cycle — each datagram is sent individually (UDP has no stream batching):

```
SendAsync(buffer)
    → HandleAsyncSend() — adds ArraySegment to _udpBatchList
    → HandleAsyncFlush()
        → foreach segment: Socket.SendToAsync(segment, Endpoint)
        → DatagramsSent++
        → _udpBatchList.Clear()
```

Unlike TCP, UDP datagrams cannot be merged — each segment in the batch is sent as a **separate `SendToAsync` call**.

---

## Disconnect

`Disconnect()` does not close the socket (the server owns it). It cancels the send loop channel, clears state, fires `OnDisconnected()`, and unregisters the session from the server registry.

---

## Inherited API

All send, receive, statistics, and lifecycle hook methods are inherited from `SessionTransport`. See [SessionTransport](transport-session.md).

---

## Remarks

- `HandleSend()` returns `SocketError.DestinationAddressRequired` if `Endpoint` is `null`.
- There is no `HandleReceive()` override — incoming data is pushed to the session by `UdpServer.ProcessReceiveLoop()` via `session.HandleReceived()`.
- `IsConnected` is set to `true` immediately when the session is created (on first datagram) and `false` on `Disconnect()`. There is no handshake.
