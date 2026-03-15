# ClientTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport`

Base class for all client-side connections. Extends `SessionTransport` with connect/reconnect logic and client-specific socket options.

```csharp
public class ClientTransport : SessionTransport
```

---

## Identity

| Property | Type | Description |
|---|---|---|
| `Address` | `string` | Server address |
| `Port` | `int` | Server port |
| `Endpoint` | `EndPoint` | Combined endpoint |
| `IsConnecting` | `bool` | `true` during async connection attempt |

---

## Connect / Disconnect

```csharp
bool Connect()
void Connect(IPAddress address, int port)
void Connect(string address, int port)
void Connect(IPEndPoint endpoint)
void Connect(DnsEndPoint endpoint)
bool Disconnect()
bool Reconnect()

bool ConnectAsync()
bool DisconnectAsync()
bool ReconnectAsync()
```

---

## Socket Options

| Property | Default | Description |
|---|---|---|
| `OptionDualMode` | `false` | IPv4+IPv6 dual mode |
| `OptionKeepAlive` | `false` | TCP keep-alive |
| `OptionNoDelay` | `false` | Disable Nagle's algorithm |
| `OptionTcpKeepAliveTime` | `-1` | Keep-alive idle time in seconds. `-1` = OS default |
| `OptionTcpKeepAliveInterval` | `-1` | Keep-alive probe interval |
| `OptionTcpKeepAliveRetryCount` | `-1` | Keep-alive probe count |

All send/receive options and statistics from `SessionTransport` are also inherited. See [SessionTransport](transport-session.md).

---

## Inherited API

All send, receive, disconnect, and lifecycle hook methods are inherited from `SessionTransport`. See [SessionTransport](transport-session.md) for the full reference.
