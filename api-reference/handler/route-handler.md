# RouteHandler & IRoutable

**Namespace:** `LuciferCore.Handler`

`RouteHandler` is the single abstract base class for all LuciferCore handlers. It owns the shared, lock-free routing table and the compiled dispatch pipeline (built via `Expression.Lambda` at startup — zero reflection at request time).

The routing system is **protocol-agnostic** — it does not care whether the incoming data is a WebSocket binary frame, an HTTP request, or any custom payload. Any data type that implements `IRoutable` can be routed. Any session that extends `SessionTransport` can carry it.

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
    [Authorize(UserRole.Guest)]
    protected void GetUser([Data] RequestModel request, [Session] HttpsSession session) { ... }

    [HttpPost("")]
    [Authorize(UserRole.Guest)]
    protected void CreateUser([Data] RequestModel request, [Session] HttpsSession session) { ... }
}
```

Both handlers extend the same `RouteHandler`. The difference is only in which `IRoutable` data type and routing attributes they use — not in the base class.

---

## Shared State

```csharp
private static readonly Utf8Map<RouteEntry> s_routes;
```

| Field | Description |
|---|---|
| `s_routes` | Lock-free UTF-8 key → `RouteEntry` map. Built once in the static constructor from all `[Handler]`-decorated classes, then frozen via `s_routes.Freeze()` before the first request can be served |

---

## Dispatch Pipeline

Every incoming routable payload flows through `RouteHandler.Route()`:

```csharp
RouteHandler.Route(data, session)
    ↓
Lucifer.Allow(session)            →  blocked if session-level rate limit exceeded
    ↓
ResolveRoute(data, session)
    ├─ BuildKey(data)              →  "METHOD:/version/prefix/path"
    ├─ s_routes.TryGetValue(key)   →  null if no matching route (Debug.Assert in dev)
    └─ CanHandle(data, session, entry)
           └─ runs entry.Middlewares in order via MiddlewareHandler.Middleware()
              → false on any middleware rejection
    ↓
RouteEntry
    ↓
entry.SyncInvoker(data, session)  /  entry.AsyncInvoker(data, session)
```

---

## RouteEntry

`RouteEntry` stores compiled dispatch delegates built once at startup via `Expression.Lambda`. Zero reflection at request time.

```csharp
public sealed class RouteEntry
{
    public UseMiddlewareAttribute[]?                    Middlewares  { get; init; }
    public Action<IRoutable, SessionTransport>?         SyncInvoker  { get; init; }
    public Func<IRoutable, SessionTransport, Task>?     AsyncInvoker { get; init; }
    public required MethodInfo Method  { get; init; }
    public required bool       IsAsync { get; init; }
}
```

| Property | Description |
|---|---|
| `Middlewares` | The `[UseMiddleware]` attributes declared on the route method, sorted by `Order`. Checked by `CanHandle()` before invocation |
| `Method` | Reflection metadata, kept on the entry for diagnostics/logging |
| `IsAsync` | `true` if the method returns `Task` |
| `SyncInvoker` | Compiled sync delegate `(IRoutable, SessionTransport) => void`. `null` for async routes |
| `AsyncInvoker` | Compiled async delegate `(IRoutable, SessionTransport) => Task`. `null` for sync routes |

> Note: invokers are bound to the handler **singleton instance** at compile time (captured as a constant in the expression tree) — they do not take the handler instance as a parameter.

---

## CanHandle

```csharp
private static bool CanHandle(IRoutable data, SessionTransport session, RouteEntry entry)
```

`CanHandle` is `private static` — it is **not** an override point on your handler subclass. It runs each middleware declared in `entry.Middlewares`, injecting the middleware attribute into `data` and delegating the actual allow/deny decision to `MiddlewareHandler.Middleware()`. If no middlewares are attached, the route is allowed by default.

To add cross-cutting logic such as IP filtering, auth checks, or session validation, write a `[Middleware]` class and attach it with `[UseMiddleware("name")]`

---

## Startup Registration

At startup (inside `RouteHandler`'s static constructor), for every `[Handler]`-decorated, non-abstract class:

1. `Lucifer.SetModelI(handler)` creates the handler singleton.
2. `RegisterRoutes(Type t)` reads `[Handler]` for `Version` and `Prefix`.
3. Enumerates all methods with `[Route]`-derived attributes (`[WsMessage]`, `[HttpGet]`, etc.) via `Lucifer.GetMethodsWithAttribute<RouteAttribute>`.
4. `BuildRouteEntry()` validates that `[Data]` parameters implement `IRoutable` and `[Session]` parameters inherit `SessionTransport`, then compiles sync/async invoker delegates via `Expression.Lambda`.
5. Assembles the dispatch key `METHOD:/{version}/{prefix}/{path}` and inserts the `RouteEntry` into `s_routes`.

After all classes are registered, `s_routes.Freeze()` is called once, locking the map for fast concurrent reads at request time.

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
    public void Handle([Data] MyPacket packet, [Session] MySession session) { ... }
}
```

---

## Remarks

- `RouteHandler` is the single base for all handlers.
- Type mismatches between `[Data]`/`[Session]` parameter declarations and actual runtime types are caught at startup; the mismatched parameter is bound to a `null` constant instead of throwing.
- `CanHandle()` is private and not overridable — extend behavior through `[Middleware]` + `[UseMiddleware]` instead.

---