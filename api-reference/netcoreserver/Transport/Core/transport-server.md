# ServerTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

`ServerTransport` is the base class for all servers.

It handles:
- accept loop
- connected session registry
- multicast
- server lifecycle (start/stop/restart)

```csharp
public class ServerTransport : IDisposable
```

---

## Construction

Constructors are internal/protected flow via subclass usage:

```csharp
internal ServerTransport(IPAddress address, int port)
internal ServerTransport(string address, int port)
internal ServerTransport(DnsEndPoint endpoint)
internal ServerTransport(IPEndPoint endpoint)
```

---

## Identity & runtime state

| Member | Type | Meaning |
|---|---|---|
| `ServerInfo.Id` | `Guid` | server unique ID |
| `ServerInfo.Port` | `int` | listen port |
| `Address` | `string` | listen address |
| `Endpoint` | `EndPoint` | actual bound endpoint |
| `IsStarted` | `bool` | server started state |
| `IsAccepting` | `bool` | accept loop active |
| `ConnectedSessions` | `long` | current connected sessions |

---

## Lifecycle methods

```csharp
virtual bool Start()
virtual bool Stop()
virtual bool Restart()
```

| Method | Behavior |
|---|---|
| `Start()` | create socket, apply options, bind, reset metrics, start accepting |
| `Stop()` | stop accept, close socket, disconnect sessions |
| `Restart()` | stop then start |

---

## Socket options applied in start

From `ServerInfo.Options`:

- `OptionReuseAddress` -> `SO_REUSEADDR`
- `OptionExclusiveAddressUse` -> `SO_EXCLUSIVEADDRUSE`
- `OptionDualMode` -> IPv4+IPv6 dual mode (IPv6 endpoint only)
- `OptionAcceptorBacklog` -> listen backlog

For deeper socket customization, override `ApplySocketOptions()` / `CreateSocket()`.

---

## Session management

```csharp
SessionTransport? FindSession(long id)
virtual bool DisconnectAll()
```

- `FindSession` returns session by ID or `null`
- `DisconnectAll` disconnects all active sessions (requires started server)

Session register/unregister is handled internally.

---

## Multicast

```csharp
virtual bool Multicast<T>(ReadOnlySpan<T> data) where T : unmanaged
```

Sends same payload to all connected sessions.

Returns `false` if:
- server not started
- payload empty

Examples:

```csharp
server.Multicast<byte>(buffer.AsSpan());
server.Multicast<char>("Hello, everyone!");
```

---

## Session factory

Override to provide custom session class:

```csharp
protected virtual SessionTransport CreateSession() => new(this);

// custom
protected override ChatSession CreateSession() => new(this);
```

---

## Lifecycle hooks (override points)

| Hook | Called when |
|---|---|
| `OnStarting` / `OnStarted` | around start |
| `OnStopping` / `OnStopped` | around stop |
| `OnConnecting` / `OnConnected` | client connect |
| `OnHandshaking` / `OnHandshaked` | handshake phase |
| `OnDisconnecting` / `OnDisconnected` | client disconnect |

---

## Server error event

```csharp
event Action<SocketError>? OnSocketError
```

Raised for server-level socket errors via internal error path.

Some expected connection-close errors are intentionally ignored.

---

## Disposal

```csharp
void Dispose()
protected virtual void Dispose(bool disposingManagedResources)
```

`Dispose()` stops server (if needed) and marks disposed state.

If overriding `Dispose(bool)`, call `base.Dispose(...)`.