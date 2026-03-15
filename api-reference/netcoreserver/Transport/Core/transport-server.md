# ServerTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport`

Base class for all servers. Manages the accept loop, session registry, multicasting, and server lifecycle.

```csharp
public class ServerTransport : IDisposable
```

---

## Identity & State

| Property | Type | Description |
|---|---|---|
| `Id` | `Guid` | Unique server identifier |
| `Address` | `string` | Listening address |
| `Port` | `int` | Listening port |
| `Endpoint` | `EndPoint` | Combined endpoint |
| `IsStarted` | `bool` | `true` when the server is running |
| `IsAccepting` | `bool` | `true` while the accept loop is active |
| `ConnectedSessions` | `long` | Number of currently connected sessions |

---

## Start / Stop

```csharp
bool Start()
bool Stop()
bool Restart()
```

---

## Socket Options

| Property | Default | Description |
|---|---|---|
| `OptionAcceptorBacklog` | `1024` | Max pending connections in accept queue |
| `OptionNoDelay` | `false` | Disable Nagle's algorithm |
| `OptionKeepAlive` | `false` | Enable TCP keep-alive |
| `OptionReuseAddress` | `false` | Allow address reuse |
| `OptionDualMode` | `false` | Enable IPv4+IPv6 dual mode |
| `OptionReceiveBufferSize` | `8192` | Per-session receive buffer |
| `OptionSendBufferSize` | `8192` | Per-session send buffer |
| `OptionReceiveBufferLimit` | `0` | Max per-session receive buffer. `0` = unlimited |
| `OptionSendBufferLimit` | `0` | Max per-session send buffer. `0` = unlimited |

---

## Session Management

```csharp
bool DisconnectAll()
SessionTransport? FindSession(long id)
```

---

## Multicast

Send data to all connected sessions simultaneously:

```csharp
bool Multicast(Buffer buffer)
bool Multicast(ReadOnlySpan<byte> data)
bool Multicast(ReadOnlySpan<char> text)
```

---

## Session Factory

Override to return your custom session type:

```csharp
protected virtual SessionTransport CreateSession()
```

```csharp
protected override ChatSession CreateSession() => new(this);
```

---

## Lifecycle Hooks

Override these in your server subclass:

| Method | When called |
|---|---|
| `OnStarting()` / `OnStarted()` | Server start |
| `OnStopping()` / `OnStopped()` | Server stop |
| `OnConnecting(session)` / `OnConnected(session)` | Client connects |
| `OnHandshaking(session)` / `OnHandshaked(session)` | Handshake phase |
| `OnDisconnecting(session)` / `OnDisconnected(session)` | Client disconnects |

---

## Events

```csharp
event Action<SocketError>? OnSocketError
```

Raised on socket errors at the server level.
