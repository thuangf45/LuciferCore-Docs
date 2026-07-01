# ServerTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

Base class for all servers. Manages the accept loop, session registry, multicasting, and server lifecycle.

```csharp
public class ServerTransport : IDisposable
```

---

## Constructing a Server

`ServerTransport` is constructed internally — subclasses inherit one of the protected constructors and pass through an address/port, endpoint, or DNS endpoint:

```csharp
internal ServerTransport(IPAddress address, int port)
internal ServerTransport(string address, int port)
internal ServerTransport(DnsEndPoint endpoint)
internal ServerTransport(IPEndPoint endpoint)
```

---

## Identity & State

Most identity/runtime fields live on `ServerInfo`, accessed via the `ServerInfo` ref property, not as flat properties on the server itself:

| Member | Type | Description |
|---|---|---|
| `ServerInfo.Id` | `Guid` | Unique server identifier, generated on construction |
| `ServerInfo.Port` | `int` | Listening port |
| `Address` | `string` | Listening address |
| `Endpoint` | `EndPoint` | Combined endpoint. Refreshed to the actual bound endpoint after `Start()` |
| `IsStarted` | `bool` | `true` when the server is running |
| `IsAccepting` | `bool` | `true` while the accept loop is active |
| `ConnectedSessions` | `long` | Number of currently connected sessions (`_sessions.Count`) |

---

## Start / Stop

```csharp
virtual bool Start()
virtual bool Stop()
virtual bool Restart()
```

| Method | Description |
|---|---|
| `Start()` | No-op if already started. Creates the acceptor socket, applies socket options, binds, refreshes `Endpoint`, resets `ServerInfo.Metric` counters, then begins accepting connections. |
| `Stop()` | No-op if not started. Stops accepting, closes/disposes the acceptor socket, disconnects all sessions. |
| `Restart()` | `Stop()` then waits for `IsStarted == false`, then `Start()`. |

---

## Socket Options

Applied in `ApplySocketOptions()`, called from `Start()` before binding:

| Option | Description |
|---|---|
| `ServerInfo.Options.OptionReuseAddress` | `SO_REUSEADDR` on the acceptor socket |
| `ServerInfo.Options.OptionExclusiveAddressUse` | `SO_EXCLUSIVEADDRUSE` on the acceptor socket |
| `ServerInfo.Options.OptionDualMode` | Enables IPv4+IPv6 dual mode — only applied when `Endpoint.AddressFamily == InterNetworkV6`, and must be set before `Listen()` |
| `ServerInfo.Options.OptionAcceptorBacklog` | Passed to `_acceptorSocket.Listen()` as the pending-connection backlog size |

> Override `ApplySocketOptions()` or `CreateSocket()` to customize socket setup (e.g. per-session buffer sizes, `NoDelay`, `KeepAlive`) — those are not applied at the server/acceptor level in this class.

---

## Session Management

```csharp
SessionTransport? FindSession(long id)
virtual bool DisconnectAll()
```

| Method | Description |
|---|---|
| `FindSession(id)` | Looks up an active session by ID from the internal `_sessions` map. Returns `null` if not found. |
| `DisconnectAll()` | Disconnects every currently connected session. Returns `false` if the server isn't started. |

Session registration/unregistration (`RegisterSession`, `UnregisterSession`) is `internal` and handled automatically as connections are accepted and torn down — assigning each session a unique, incrementing `SessionInfo.Id`.

---

## Multicast

Send data to all connected sessions simultaneously:

```csharp
virtual bool Multicast<T>(ReadOnlySpan<T> data) where T : unmanaged
```

A single generic overload — works for any unmanaged element type (`byte`, `char`, etc. via `ReadOnlySpan<T>`). Returns `false` if the server isn't started or `data` is empty.

```csharp
server.Multicast<byte>(buffer.AsSpan());
server.Multicast<char>("Hello, everyone!");
```

---

## Session Factory

Override to return your custom session type:

```csharp
protected virtual SessionTransport CreateSession() => new(this);
```

```csharp
protected override ChatSession CreateSession() => new(this);
```

---

## Lifecycle Hooks

Override these `protected virtual` methods in your server subclass. Each is invoked through an internal wrapper (`OnConnectingInternal`, etc.) called by the transport layer:

| Method | When called |
|---|---|
| `OnStarting()` / `OnStarted()` | Around `Start()` — `OnStarted()` defaults to logging `"Started"` |
| `OnStopping()` / `OnStopped()` | Around `Stop()` — `OnStopped()` defaults to logging `"Stopped"` |
| `OnConnecting(session)` / `OnConnected(session)` | Client connects |
| `OnHandshaking(session)` / `OnHandshaked(session)` | Handshake phase |
| `OnDisconnecting(session)` / `OnDisconnected(session)` | Client disconnects — `OnDisconnectedInternal` also unregisters the session from `_sessions` before calling `OnDisconnected()` |

---

## Events

```csharp
event Action<SocketError>? OnSocketError
```

Raised via `SendError()` on socket errors at the server level. Connection-teardown errors (`ConnectionAborted`, `ConnectionRefused`, `ConnectionReset`, `OperationAborted`, `Shutdown`) are intentionally swallowed and never raise the event.

---

## Disposal

```csharp
void Dispose()
protected virtual void Dispose(bool disposingManagedResources)
```

Standard dispose pattern. `Dispose()` calls `Stop()` if the server hasn't already been disposed, then marks `IsDisposed = true`. Override `Dispose(bool)` for custom cleanup, always calling `base.Dispose(disposing)`.

---