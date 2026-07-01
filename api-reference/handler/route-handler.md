# RouteHandler & IRoutable

**Namespace:** `LuciferCore.Handler`

`RouteHandler` is the base class for all handlers in LuciferCore.

Routing is protocol-agnostic:
- works with WebSocket
- works with HTTP
- works with custom protocols

If data implements `IRoutable`, it can be routed.  
If session inherits `SessionTransport`, it can carry routed data.

```csharp
public abstract class RouteHandler
```

---

## `IRoutable`

```csharp
public interface IRoutable
{
    ReadOnlySpan<byte> MethodRoute { get; }
    ReadOnlySpan<byte> UrlRoute    { get; }
}
```

`IRoutable` gives routing info from payload.

| Property | Meaning |
|---|---|
| `MethodRoute` | Method key (`MSG`, `GET`, `POST`, custom key) |
| `UrlRoute` | Route path (`/v1/wss/ChatMessage`, `/v1/api/user`, ...) |

Built-in types:
- `PacketModel`
- `RequestModel`

You can create your own type by implementing `IRoutable`.

---

## Extend `RouteHandler`

```csharp
[Handler("v1", "wss")]
internal class ChatHandler : RouteHandler
{
    [Message("ChatMessage")]
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

Both use same base class.  
Difference is route attributes + data type.

---

## Shared route table

```csharp
private static readonly Utf8Map<RouteEntry> s_routes;
```

` s_routes ` stores route key -> route entry.  
Built once at startup, then frozen for fast concurrent reads.

---

## Dispatch flow (simple)

```text
RouteHandler.Route(data, session)
 -> session-level checks
 -> build key from MethodRoute + UrlRoute
 -> find route in s_routes
 -> run middlewares by order
 -> call sync or async handler delegate
```

No per-request reflection.

---

## `RouteEntry`

Each route stores prebuilt delegates.

```csharp
public sealed class RouteEntry
{
    public UseMiddlewareAttribute[]?                Middlewares  { get; init; }
    public Action<IRoutable, SessionTransport>?     SyncInvoker  { get; init; }
    public Func<IRoutable, SessionTransport, Task>? AsyncInvoker { get; init; }
    public required MethodInfo Method  { get; init; }
    public required bool       IsAsync { get; init; }
}
```

| Property | Meaning |
|---|---|
| `Middlewares` | Method middlewares sorted by `Order` |
| `SyncInvoker` | Delegate for sync method |
| `AsyncInvoker` | Delegate for async method |
| `Method` | Method metadata |
| `IsAsync` | `true` if method returns `Task` |

---

## Middleware and `CanHandle`

Route middleware is checked before handler method runs.

If any middleware returns `false`, request stops.

To add custom checks:
1. Create `[Middleware("Name")]`
2. Apply with `[UseMiddleware("Name")]`

---

## Startup registration (summary)

At startup, LuciferCore:

1. Finds all `[Handler]` classes
2. Reads route attributes (`[Message]`, `[HttpGet]`, ...)
3. Validates `[Data]` and `[Session]` parameter types
4. Builds delegates
5. Inserts route entries into `s_routes`
6. Freezes route map

---

## Custom `IRoutable`

```csharp
public class MyPacket : IRoutable
{
    public string Action { get; set; } = "";
    public byte[] Payload { get; set; } = [];

    public ReadOnlySpan<byte> MethodRoute => "MSG"u8;
    public ReadOnlySpan<byte> UrlRoute => Encoding.UTF8.GetBytes($"/v1/myprotocol/{Action}");
}
```

```csharp
[Handler("v1", "myprotocol")]
internal class MyHandler : RouteHandler
{
    [Message("Action")]
    public void Handle([Data] MyPacket packet, [Session] MySession session) { ... }
}
```

---

## Notes

- `RouteHandler` is the single handler base.
- Use middleware for cross-cutting logic.
- Type mismatch in `[Data]`/`[Session]` is validated during startup wiring.
- Routing system is open: you can extend route style and `IRoutable` payload types.