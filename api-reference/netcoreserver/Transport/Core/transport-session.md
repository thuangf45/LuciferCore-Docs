# SessionTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

Base session transport for handling client connections with async send/receive.

```csharp
public class SessionTransport : IDisposable
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Server` | `ServerTransport` | Parent server instance |
| `Socket` | `Socket` | Underlying network socket |
| `Cache` | `Buffer` | Session cache buffer |
| `SessionInfo` | `ref SessionInfo` | Session information struct reference (options, metrics, id, etc.) |
| `IsConnected` | `bool` | True if session connected |
| `IsDisposed` | `bool` | True if session disposed |
| `IsSocketDisposed` | `bool` | True if socket disposed |

## Events

```csharp
event Action<SocketError>? OnSocketError
```

Socket error event handler.

---

## Send API

```csharp
long Send<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendAsync<T>(ReadOnlySpan<T> data) where T : unmanaged
```

Sends generic data synchronously/asynchronously. Supports `byte` and `char` spans directly; other unmanaged types are sent as their byte representation.

## Receive API

```csharp
long   Receive(byte[] buffer)
long   Receive(byte[] buffer, long offset, long size)
string Receive(long size)   // receives and decodes as UTF-8 text
```

## Disconnect / Dispose

```csharp
bool Disconnect()
void Dispose()
```

---

## Lifecycle Hooks

Protected, overridable in a derived session class:

| Method | When called |
|---|---|
| `OnConnecting()` | Before session connects |
| `OnConnected()` | After session connected |
| `OnHandshaking()` | During handshaking |
| `OnHandshaked()` | After handshake completed |
| `OnDisconnecting()` | Before session disconnects |
| `OnDisconnected()` | After session disconnected |
| `OnReceived(byte[] buffer, long offset, long size)` | When data received |
| `OnSent(long sent, long pending)` | When data sent |
| `OnEmpty()` | When send queue is empty |
| `Dispose(bool disposingManagedResources)` | Dispose pattern implementation |
| `ApplySocketOptions()` | Applies socket options from server configuration (keep-alive, no-delay) |
| `TryReceive()` | Attempts async receive if not already receiving |
| `TrySend()` | Attempts to send pending data |

---