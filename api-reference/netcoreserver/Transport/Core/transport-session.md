# SessionTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

`SessionTransport` is the base session class for client connection handling.

It provides:
- socket ownership
- async send/receive flow
- lifecycle hooks
- session metrics/options access

```csharp
public class SessionTransport : IDisposable
```

---

## Properties

| Property | Type | Meaning |
|---|---|---|
| `Server` | `ServerTransport` | parent server |
| `Socket` | `Socket` | underlying socket |
| `Cache` | `Buffer` | session buffer cache |
| `SessionInfo` | `ref SessionInfo` | session metadata/options/metrics |
| `IsConnected` | `bool` | connected state |
| `IsDisposed` | `bool` | session disposed state |
| `IsSocketDisposed` | `bool` | socket disposed state |

---

## Event

```csharp
event Action<SocketError>? OnSocketError
```

Raised on socket-level errors for this session.

---

## Send API

```csharp
long Send<T>(ReadOnlySpan<T> data) where T : unmanaged
bool SendAsync<T>(ReadOnlySpan<T> data) where T : unmanaged
```

- Sync and async send paths
- Supports `byte` and `char` spans directly
- Other unmanaged `T` are sent as raw byte representation

---

## Receive API

```csharp
long   Receive(byte[] buffer)
long   Receive(byte[] buffer, long offset, long size)
string Receive(long size)
```

- Byte receive overloads
- UTF-8 decoded string receive overload

---

## Disconnect / dispose

```csharp
bool Disconnect()
void Dispose()
```

- `Disconnect()` closes active connection
- `Dispose()` releases session resources

---

## Lifecycle hooks (override points)

| Method | Called when |
|---|---|
| `OnConnecting()` | before connect |
| `OnConnected()` | after connect |
| `OnHandshaking()` | during handshake |
| `OnHandshaked()` | after handshake |
| `OnDisconnecting()` | before disconnect |
| `OnDisconnected()` | after disconnect |
| `OnReceived(byte[] buffer, long offset, long size)` | data received |
| `OnSent(long sent, long pending)` | data sent |
| `OnEmpty()` | send queue becomes empty |
| `Dispose(bool disposingManagedResources)` | dispose pipeline |
| `ApplySocketOptions()` | apply session socket options |
| `TryReceive()` | attempt async receive |
| `TrySend()` | attempt sending pending data |

---

Use derived session classes to implement protocol-specific behavior through these hooks.