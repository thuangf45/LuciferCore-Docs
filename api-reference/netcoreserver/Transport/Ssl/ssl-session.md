# SslSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.SSL`

`SslSession` is the server-side TLS/SSL session class.  
It extends `SessionTransport` and uses `SslStream` for encrypted I/O.

```csharp
public class SslSession : SessionTransport
```

---

## Constructor

```csharp
public SslSession(SslServer server)
```

Created by `SslServer.CreateSession()`.  
You usually do not create it manually.

---

## Handshake state

| Property | Type | Description |
|---|---|---|
| `IsHandshaked` | `bool` | `true` after TLS handshake is done |

Before handshake completes, send/receive calls are rejected.

---

## Handshake flow (simple)

1. client connects
2. server starts TLS handshake
3. `OnHandshaking()` hook is called
4. handshake completes
5. `IsHandshaked = true`
6. receive loop starts
7. `OnHandshaked()` hook is called

If handshake fails, session reports error and disconnects.

---

## Send / Receive behavior

After handshake:

- data is encrypted/decrypted by `SslStream`
- public send/receive APIs work as normal

Before handshake:

- send returns `false`/`0`
- receive returns `0`

---

## Hooks you can override

Use lifecycle hooks from the base class, especially:

- `OnHandshaking()`
- `OnHandshaked()`
- other session hooks (`OnConnected`, `OnDisconnected`, `OnReceived`, `OnSent`, ...)

---

## Inherited API

`SslSession` adds TLS behavior.  
Other APIs are inherited from `SessionTransport`, including:

- `Send<T>(...)`, `SendAsync<T>(...)`
- `Receive(...)`
- `Disconnect()`
- lifecycle hooks/events
- metrics/options access
- `Dispose()`

---

## Notes

- `SslSession` uses the parent `SslServer.Context` (certificate/protocol settings).
- `SslStream` owns the inner network stream in this implementation.