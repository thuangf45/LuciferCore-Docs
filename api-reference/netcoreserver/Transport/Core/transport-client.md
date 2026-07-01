# ClientTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

Base class for all client-side connections. Extends `SessionTransport` with connect/reconnect logic and client-specific socket setup.

```csharp
public class ClientTransport : SessionTransport
```

---

## Construction

`ClientTransport` is built via `internal` constructors — it's not meant to be instantiated directly with arbitrary address/port overloads at the public API surface; a derived/factory type is expected to expose the public client API. The internal constructors accept:

```csharp
internal ClientTransport(IPAddress address, int port)
internal ClientTransport(string address, int port)
internal ClientTransport(IPEndPoint endpoint)
internal ClientTransport(DnsEndPoint endpoint)   // resolves DNS, prefers IPv4
internal ClientTransport(EndPoint endpoint, string address, int port) // main constructor
```

For `DnsEndPoint`, the original hostname is preserved in `Address` (important for SSL/TLS certificate validation), while `Endpoint` is resolved to a concrete `IPEndPoint` (IPv4 preferred if available).

---

## Identity

| Property | Type | Description |
|---|---|---|
| `Address` | `string` | Server address (or original hostname, for DNS-based construction) |
| `Port` | `int` | Server port |
| `Endpoint` | `EndPoint` | Resolved network endpoint used to connect |
| `IsConnecting` | `bool` | `true` while an async connection attempt is in progress |

---

## Connect / Disconnect

```csharp
bool Connect()          // synchronous; uses Address/Port/Endpoint set at construction
bool Disconnect()        // override, delegates to SessionTransport.Disconnect()
bool Reconnect()         // Disconnect() then Connect()

bool ConnectAsync()
bool DisconnectAsync()   // currently synchronous: calls Disconnect() under the hood
bool ReconnectAsync()    // DisconnectAsync(), busy-waits (Thread.Yield) until !IsConnected, then ConnectAsync()
```

There is no parameterized public `Connect(address, port)` overload — the target endpoint is fixed at construction time via the internal constructors above.

`Connect()` performs, in order: rent receive/send buffers, set up event args, create the socket (`CreateSocket()`), apply dual-mode setup (`ClientSetUp()`), fire `OnConnecting()`, attempt the connection (`TryConnect()`), apply socket options (`ApplySocketOptions()`), reserve buffer capacity, fire `OnConnected()`, start the first receive (`FirstTryReceive()`), then run `HandleHandshake()`. If anything along the way fails, `Connect()` returns `false` early.

---

## Socket Options

These are not properties on `ClientTransport` itself — they live on `SessionInfo.Options` and are applied by `ClientSetUp()` / `ApplySocketOptions()` during connect:

| `SessionInfo.Options` member | Description |
|---|---|
| `OptionDualMode` | IPv4+IPv6 dual mode; applied only when `Socket.AddressFamily == InterNetworkV6`, before connecting |
| `OptionKeepAlive` | Enables TCP keep-alive |
| `OptionTcpKeepAliveTime` | Keep-alive idle time; applied if `>= 0` |
| `OptionTcpKeepAliveInterval` | Keep-alive probe interval; applied if `>= 0` |
| `OptionTcpKeepAliveRetryCount` | Keep-alive probe count; applied if `>= 0` |
| `OptionNoDelay` | Disables Nagle's algorithm when `true` |

All send/receive options and statistics from `SessionTransport` are also inherited. See [SessionTransport](transport-session.md).

---

## Overridable Members

For subclasses needing custom connection behavior (e.g. an SSL/TLS client):

```csharp
protected virtual Socket CreateSocket()              // default: new Socket(Endpoint.AddressFamily, SocketType.Stream, ProtocolType.Tcp)
protected virtual void ClientSetUp()                  // applies OptionDualMode for IPv6 sockets
protected virtual bool TryConnect()                   // synchronous Socket.Connect(Endpoint); cleans up and fires OnDisconnected() on SocketException
protected override void ApplySocketOptions()          // applies keep-alive / no-delay options from SessionInfo.Options
internal override void ProcessConnect(SocketAsyncEventArgs e)  // completion handler for ConnectAsync(); applies options, fires OnConnected(), runs HandleHandshake(), disconnects on handshake failure
```

---

## Inherited API

All send, receive, disconnect, and lifecycle hook methods (`OnConnecting`, `OnConnected`, `OnDisconnecting`, `OnDisconnected`, `OnEmpty`, `HandleHandshake`, etc.) are inherited from `SessionTransport`. See `SessionTransport` for the full reference.

---