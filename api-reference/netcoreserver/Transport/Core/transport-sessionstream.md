# StreamSessionTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

`StreamSessionTransport` is the default TCP-style server session transport.

It is a concrete implementation of `SessionTransport` for stream sockets.

```csharp
public class StreamSessionTransport : SessionTransport
```

---

## Construction

```csharp
public StreamSessionTransport(ServerTransport server)
```

Creates a session transport bound to its parent server.

---

## Public surface

`StreamSessionTransport` does not add extra public methods beyond construction.

Use inherited APIs from `SessionTransport`, including:

- `Send<T>(ReadOnlySpan<T> data)`
- `SendAsync<T>(ReadOnlySpan<T> data)`
- `Receive(...)`
- `Disconnect()`
- `Dispose()`
- lifecycle hooks/events from the base session

---

## Protected extensibility

This class follows the `SessionTransport` lifecycle model.

For customization, override protected hooks in your derived session class (for example: `OnReceived`, `OnSent`, `OnConnected`, `OnDisconnected`, `OnEmpty`, `Dispose(bool)`).

---

## Notes

- Use this class when you want standard stream-based server session behavior.
- Put protocol/app logic in your own derived session type.