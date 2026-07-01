# Socket Info Struct

**Namespace:** `LuciferCore.View`

Plain data structs holding server/session identity, network metrics, and socket configuration.

---

## ServerInfo

```csharp
public struct ServerInfo
```

Represents server information.

| Field | Type | Description |
|---|---|---|
| `Id` | `Guid` | Unique server ID |
| `Port` | `int` | Server port number |
| `Metric` | `MetricNetwork` | Network metrics for the server |
| `Options` | `SocketOptions` | Socket options for the server |

```csharp
public ServerInfo()        // constructs and initializes
public void Initialize()   // resets Metric and Options to defaults
```

---

## SessionInfo

```csharp
public struct SessionInfo
```

Represents session information.

| Field | Type | Description |
|---|---|---|
| `Id` | `long` | Unique session ID |
| `Metric` | `MetricNetwork` | Network metrics for the session |
| `Options` | `SocketOptions` | Socket options for the session |

```csharp
public SessionInfo()       // constructs and initializes
public void Initialize()   // resets Metric and Options to defaults
```

---

## MetricNetwork

```csharp
public struct MetricNetwork
```

Represents network metrics for a server or session.

| Field | Type | Description |
|---|---|---|
| `BytesPending` | `long` | Bytes pending to send |
| `BytesSending` | `long` | Bytes currently being sent |
| `BytesSent` | `long` | Total bytes sent |
| `BytesReceived` | `long` | Total bytes received |
| `DatagramsReceived` | `long` | Total datagrams received by server |

```csharp
public MetricNetwork()     // constructs and initializes (all fields = 0)
public void Initialize()   // resets all fields to 0
```

---

## SocketOptions

```csharp
public struct SocketOptions
```

Represents socket options.

| Field | Type | Default | Description |
|---|---|---|---|
| `OptionAcceptorBacklog` | `int` | `1024` | Acceptor backlog size |
| `OptionDualMode` | `bool` | `false` | Dual-mode socket (IPv4 and IPv6 support) |
| `OptionKeepAlive` | `bool` | `false` | SO_KEEPALIVE socket option |
| `OptionTcpKeepAliveTime` | `int` | `-1` | Keep-alive time in seconds (-1 = disabled) |
| `OptionTcpKeepAliveInterval` | `int` | `-1` | Keep-alive interval in seconds (-1 = disabled) |
| `OptionTcpKeepAliveRetryCount` | `int` | `-1` | Keep-alive retry count (-1 = disabled) |
| `OptionNoDelay` | `bool` | `false` | NODELAY (disables Nagle's algorithm) |
| `OptionReuseAddress` | `bool` | `false` | SO_REUSEADDR socket option |
| `OptionExclusiveAddressUse` | `bool` | `false` | SO_EXCLUSIVEADDRUSE socket option |
| `OptionReceiveBufferSize` | `int` | `8192` | Receive buffer size in bytes |
| `OptionSendBufferSize` | `int` | `8192` | Send buffer size in bytes |
| `OptionReceiveBufferLimit` | `int` | `0` | Receive buffer limit (0 = unlimited) |
| `OptionSendBufferLimit` | `int` | `0` | Send buffer limit (0 = unlimited) |

```csharp
public SocketOptions()     // constructs and initializes with default values
public void Initialize()   // resets all fields to the default values above
```

---