# Socket Info Struct

**Namespace:** `LuciferCore.View`

These are plain data structs for:
- server/session identity
- network metrics
- socket options

---

## `ServerInfo`

```csharp
public struct ServerInfo
```

Represents server-level info.

| Field | Type | Meaning |
|---|---|---|
| `Id` | `Guid` | server unique ID |
| `Port` | `int` | server port |
| `Metric` | `MetricNetwork` | server network metrics |
| `Options` | `SocketOptions` | server socket options |

Methods:

```csharp
public ServerInfo()
public void Initialize()
```

`Initialize()` resets `Metric` and `Options` to defaults.

---

## `SessionInfo`

```csharp
public struct SessionInfo
```

Represents session-level info.

| Field | Type | Meaning |
|---|---|---|
| `Id` | `long` | session unique ID |
| `Metric` | `MetricNetwork` | session network metrics |
| `Options` | `SocketOptions` | session socket options |

Methods:

```csharp
public SessionInfo()
public void Initialize()
```

`Initialize()` resets `Metric` and `Options` to defaults.

---

## `MetricNetwork`

```csharp
public struct MetricNetwork
```

Network counters for server/session.

| Field | Type | Meaning |
|---|---|---|
| `BytesPending` | `long` | pending send bytes |
| `BytesSending` | `long` | bytes currently sending |
| `BytesSent` | `long` | total sent bytes |
| `BytesReceived` | `long` | total received bytes |
| `DatagramsReceived` | `long` | total received datagrams (server) |

Methods:

```csharp
public MetricNetwork()
public void Initialize()
```

`Initialize()` sets all counters to `0`.

---

## `SocketOptions`

```csharp
public struct SocketOptions
```

Socket configuration options.

| Field | Type | Default | Meaning |
|---|---|---|---|
| `OptionAcceptorBacklog` | `int` | `1024` | accept backlog |
| `OptionDualMode` | `bool` | `false` | IPv4+IPv6 dual mode |
| `OptionKeepAlive` | `bool` | `false` | keepalive enabled |
| `OptionTcpKeepAliveTime` | `int` | `-1` | keepalive time (sec) |
| `OptionTcpKeepAliveInterval` | `int` | `-1` | keepalive interval (sec) |
| `OptionTcpKeepAliveRetryCount` | `int` | `-1` | keepalive retry count |
| `OptionNoDelay` | `bool` | `false` | disable Nagle |
| `OptionReuseAddress` | `bool` | `false` | SO_REUSEADDR |
| `OptionExclusiveAddressUse` | `bool` | `false` | SO_EXCLUSIVEADDRUSE |
| `OptionReceiveBufferSize` | `int` | `8192` | recv buffer size |
| `OptionSendBufferSize` | `int` | `8192` | send buffer size |
| `OptionReceiveBufferLimit` | `int` | `0` | recv limit (`0` = unlimited) |
| `OptionSendBufferLimit` | `int` | `0` | send limit (`0` = unlimited) |

Methods:

```csharp
public SocketOptions()
public void Initialize()
```

`Initialize()` restores all defaults above.