# UdsSession

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDS`

`UdsSession` is the Unix Domain Socket server session class.  
It extends `StreamSessionTransport`.

```csharp
public class UdsSession : StreamSessionTransport
```

---

## Constructor

```csharp
public UdsSession(UdsServer server)
```

Created by `UdsServer.CreateSession()`.  
You usually do not create it manually.

---

## Socket options

`UdsSession` overrides `ApplySocketOptions()` as empty.  
TCP-specific socket options are not used for UDS.

---

## Inherited API

`UdsSession` adds no new public methods.

Use inherited APIs from `StreamSessionTransport` / `SessionTransport`, such as:

- `Send<T>(...)`
- `SendAsync<T>(...)`
- `Receive(...)`
- `Disconnect()`
- `Dispose()`
- lifecycle hooks/events

---

## Notes

- Use `UdsSession` for UDS server connections.
- Put protocol/app logic in your own derived session class.