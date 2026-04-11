# RouteHandler & IRoutable

**Namespace:** `LuciferCore.Handler`

`RouteHandler` is the single abstract base class for all LuciferCore handlers. It owns the shared, lock-free routing table, the role cache, and the compiled dispatch pipeline.

The routing system is now **protocol-agnostic** — it does not care whether the incoming data is a WebSocket binary frame, an HTTP request, or any custom payload. Any data type that implements `IRoutable` can be routed. Any session that extends `SessionTransport` can carry it.

```csharp
public abstract class RouteHandler
```

---

## IRoutable

```csharp
public interface IRoutable
{
    ReadOnlySpan<byte> MethodRoute { get; }
    ReadOnlySpan<byte> UrlRoute    { get; }
}
```

The contract any data payload must implement to participate in routing. The dispatch pipeline reads these two UTF-8 byte spans to build the route lookup key `METHOD:/path`.

| Property | Description |
|---|---|
| `MethodRoute` | The method token as a UTF-8 span — e.g. `MSG`, `GET`, `POST`, or any custom verb |
| `UrlRoute` | The full URL/path as a UTF-8 span — e.g. `/v1/wss/ChatMessage`, `/v1/api/user` |

Both `PacketModel` and `RequestModel` implement `IRoutable`. Any custom data type can be routed by implementing this interface.

---

## Extending RouteHandler

Extend `RouteHandler` directly and decorate with `[Handler]`:

```csharp
[Handler("v1", "wss")]
internal class ChatHandler : RouteHandler
{
    [WsMessage("ChatMessage")]
    [Safe("")]
    [RateLimiter(20, 1)]
    [Authorize(UserRole.Guest)]
    public void SendChat([Session] ChatSession session, [Data] PacketModel data)
    {
        using var _ = data;
        ((WssServer)session.Server).MulticastBinary(data.Buffer);
    }
}
```

```csharp
[Handler("v1", "/api/user")]
internal class UserHandler : RouteHandler
{
    [HttpGet("")]
    [Safe("")]
    [Authorize(UserRole.Guest)]
    protected void GetUser([Data] RequestModel request, [Session] HttpsSession session) { ... }

    [HttpPost("")]
    [Safe("")]
    [Authorize(UserRole.Guest)]
    protected void CreateUser([Data] RequestModel request, [Session] HttpsSession session) { ... }
}
```

Both handlers extend the same `RouteHandler`. The difference is only in which `IRoutable` data type and routing attributes they use — not in the base class.

---

## Shared State

```csharp
protected static readonly Utf8Map<RouteEntry>                           Routes;
protected static readonly ConcurrentDictionary<MethodInfo, UserRole>    RoleCache;
```

| Field | Description |
|---|---|
| `Routes` | Lock-free UTF-8 key → `RouteEntry` map. Built once at startup from all `[Handler]`-decorated classes. Frozen by `Optimize()` before the first request |
| `RoleCache` | Per-method role requirement cache. Populated lazily on first access using `[Authorize]` metadata |

Both fields are `static` and shared across all handler instances regardless of protocol.

---

## Dispatch Pipeline

Every incoming routable payload flows through `ResolveRoute()`:

```
IRoutable.MethodRoute + IRoutable.UrlRoute
    ↓
BuildKey(data)  →  "METHOD:/version/prefix/path"
    ↓
Routes.TryGetValue(key)        →  404 error if not found
    ↓
CanHandle(data, session, method)
    ├─ Lucifer.Allow(method)   →  429 Too Many Requests if rate-limited
    └─ RoleCache / [Authorize] →  403 Forbidden if role insufficient
    ↓
RouteEntry  (returned to caller)
    ↓
SyncInvoker / AsyncInvoker
```

---

## RouteEntry

`RouteEntry` stores compiled dispatch delegates built once at startup via `Expression.Lambda`. Zero reflection at request time.

```csharp
public sealed class RouteEntry
{
    public Action<RouteHandler, IRoutable, SessionTransport>?          SyncInvoker  { get; init; }
    public Func<RouteHandler, IRoutable, SessionTransport, Task>?      AsyncInvoker { get; init; }
    public required MethodInfo Method  { get; init; }
    public required bool       IsAsync { get; init; }
}
```

| Property | Description |
|---|---|
| `Method` | Reflection metadata. Used for rate limiting and role checks |
| `IsAsync` | `true` if the method returns `Task` |
| `SyncInvoker` | Compiled sync delegate. `null` for async routes |
| `AsyncInvoker` | Compiled async delegate. `null` for sync routes |

---

## CanHandle

```csharp
public virtual bool CanHandle(IRoutable data, SessionTransport session, MethodInfo method)
```

Runs rate limit and authorization checks. Returns `false` and sends an error response if either check fails. Override to add custom cross-cutting logic such as IP filtering or session validation.

---

## Startup Registration

At startup, `RouteHandler.RegisterRoutes(Type t)` is called for every `[Handler]`-decorated class:

1. Reads `[Handler]` for `version` and `prefix`.
2. Enumerates all methods with routing attributes (`[WsMessage]`, `[HttpGet]`, etc.).
3. Assembles the dispatch key: `METHOD:/{version}/{prefix}/{path}`.
4. Compiles sync/async invoker delegates via `Expression.Lambda`.
5. Inserts the `RouteEntry` into `Routes`.

After all classes are registered, `RouteHandler.Optimize()` freezes `Routes`.

---

## Custom IRoutable

Any data type can participate in routing by implementing `IRoutable`:

```csharp
public class MyPacket : IRoutable
{
    public string Action { get; set; } = "";
    public byte[] Payload { get; set; } = [];

    // Route key: "MSG:/v1/myprotocol/Action"
    public ReadOnlySpan<byte> MethodRoute => "MSG"u8;
    public ReadOnlySpan<byte> UrlRoute    => Encoding.UTF8.GetBytes($"/v1/myprotocol/{Action}");
}
```

Then handle it exactly like any other routable type:

```csharp
[Handler("v1", "myprotocol")]
internal class MyHandler : RouteHandler
{
    [WsMessage("Action")]
    [Safe("")]
    public void Handle([Data] MyPacket packet, [Session] MySession session) { ... }
}
```

---

## Remarks

- There are no longer protocol-specific handler base classes (`WssHandlerBase`, `WsHandlerBase`, `HttpHandlerBase`, `HttpsHandlerBase`). `RouteHandler` is the single base for all handlers.
- Type mismatches between `[Data]`/`[Session]` parameter types and actual runtime types are caught at startup via `Debug.Assert` and the mismatched parameter receives `null`.
- `CanHandle()` is `virtual` — override it for custom cross-cutting logic.
- Query strings are stripped from `UrlRoute` automatically during key construction (everything after `?`).
