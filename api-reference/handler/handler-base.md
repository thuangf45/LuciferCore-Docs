# HandlerBase

**Namespace:** `LuciferCore.Handler`

`HandlerBase` is the abstract root class for all LuciferCore handler types. It owns the shared, lock-free routing table, the role cache, and the compiled dispatch pipeline. You never extend `HandlerBase` directly — use `WssHandlerBase`, `WsHandlerBase`, `HttpsHandlerBase`, or `HttpHandlerBase` instead.

---

## Class Hierarchy

```
HandlerBase
├── WsHandlerBase    (WebSocket, unencrypted)
├── WssHandlerBase   (WebSocket Secure / TLS)
├── HttpHandlerBase  (HTTP, unencrypted)
└── HttpsHandlerBase (HTTPS / TLS)
```

Each concrete base class implements `IHandler<TData, TSession>` for its specific protocol pair.

---

## IHandler\<TData, TSession\>

```csharp
public interface IHandler<TData, TSession>
{
    RouteEntry? Handle(TData data, TSession session);
}
```

The single-method interface that every handler base class implements. `Handle()` is called by the dispatch pipeline to resolve and invoke the correct method for an incoming message or request.

| Type Parameter | Description |
|---|---|
| `TData` | The incoming payload type (`PacketModel` for WebSocket, `RequestModel` for HTTP) |
| `TSession` | The active session type (`WsSession`, `WssSession`, `HttpSession`, or `HttpsSession`) |

The return value is a `RouteEntry?` — `null` means the request was rejected (rate-limited, unauthorized, or unrouted) and an error response has already been sent.

---

## Shared State

```csharp
protected static readonly Utf8Map<RouteEntry> Routes;
protected static readonly ConcurrentDictionary<MethodInfo, UserRole> RoleCache;
```

| Field | Description |
|---|---|
| `Routes` | Lock-free UTF-8 key → `RouteEntry` map. Built once at startup from all `[Handler]`-decorated classes. Frozen by `Optimize()` before the first request |
| `RoleCache` | Per-method role requirement cache. Populated lazily on first access using `[Authorize]` metadata |

Both fields are `static` and shared across all handler instances.

---

## Dispatch Pipeline

Every incoming message flows through the following stages inside `ResolveRoute()`:

```
BuildKey(data)
    ↓
Routes.TryGetValue(key)   →  404 Error if not found
    ↓
CanHandle(data, session, method)
    ├─ Lucifer.Allow(method)     →  429 Too Many Requests if rate-limited
    └─ RoleCache / [Authorize]   →  403 Forbidden if role insufficient
    ↓
RouteEntry  (returned to concrete Handle())
    ↓
SyncInvoker / AsyncInvoker
```

---

## RouteEntry

`RouteEntry` is a sealed nested class that stores the compiled dispatch delegates for a single route. It is built once at startup using `Expression.Lambda` compilation and reused for every invocation — no reflection at request time.

```csharp
public sealed class RouteEntry
{
    public Action<HandlerBase, object, object>? SyncInvoker  { get; init; }
    public Func<HandlerBase, object, object, Task>? AsyncInvoker { get; init; }
    public required MethodInfo Method  { get; init; }
    public required bool IsAsync       { get; init; }
}
```

| Property | Type | Description |
|---|---|---|
| `Method` | `MethodInfo` | Reflection metadata for the route method. Used for rate limiting and role checks |
| `IsAsync` | `bool` | `true` if the method returns `Task`; `false` for synchronous methods |
| `SyncInvoker` | `Action<HandlerBase, object, object>?` | Compiled sync delegate. `null` for async routes |
| `AsyncInvoker` | `Func<HandlerBase, object, object, Task>?` | Compiled async delegate. `null` for sync routes |

---

## Overridable Methods

These are the extension points exposed to concrete handler bases and your application code:

| Method | Description |
|---|---|
| `BuildKey<TData>(TData data)` | Constructs the UTF-8 route lookup key from the incoming payload. Each concrete base overrides this for its protocol |
| `CanHandle<TData, TSession>(...)` | Runs the rate-limit and authorization checks. Returns `false` and sends an error response if either check fails |
| `Error<TSession, TData>(...)` | Sends an error response back to the client. Each concrete base overrides this with protocol-appropriate encoding |
| `Ok<TSession>(...)` | Sends a success response. Used by HTTP bases |

---

## Startup Registration

At startup, `HandlerBase.RegisterRoutes(Type t)` is called for every `[Handler]`-decorated class. It:

1. Reads `[Handler]` for the `version` and `prefix`.
2. Enumerates all methods decorated with `[Route]` or any routing attribute.
3. Assembles the full dispatch key: `METHOD:/{version}/{prefix}/{path}`.
4. Compiles sync/async invoker delegates via `Expression.Lambda`.
5. Inserts the resulting `RouteEntry` into the shared `Routes` map.

After all classes are registered, `HandlerBase.Optimize()` freezes the `Routes` map into a read-only, lock-free structure.

---

## Remarks

- Do not extend `HandlerBase` directly. Always use one of the four concrete base classes for your protocol.
- `RouteEntry` delegates are compiled once per route at startup. Subsequent invocations execute at near-native speed with no allocation.
- Type mismatches between `[Data]`/`[Session]` parameter types and the `IHandler<TData, TSession>` type arguments are caught at startup and logged as `ERROR`. The mismatched parameter receives `null`.
- `CanHandle()` is `virtual` — you can override it for custom cross-cutting logic such as IP filtering or session validation.
