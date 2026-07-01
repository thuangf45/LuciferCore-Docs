# MiddlewareHandler

**Namespace:** `LuciferCore.Handler`

`MiddlewareHandler` is the abstract base class for all middleware. It owns the shared, lock-free middleware registry and the compiled dispatch delegate for each registered middleware.

```csharp
public abstract class MiddlewareHandler
```

---

## Shared State

```csharp
private static readonly Utf8Map<Func<IRoutable, SessionTransport, bool>> s_compiledMiddlewares;
```

| Field | Description |
|---|---|
| `s_compiledMiddlewares` | Lock-free UTF-8 key (`[Middleware]` name) → compiled `Handle` delegate map. Built once in the static constructor, then frozen via `s_compiledMiddlewares.Freeze()` |

---

## Handle

```csharp
protected abstract bool Handle(IRoutable data, SessionTransport session);
```

The method every `[Middleware]` subclass must implement. Return `true` to allow the request to proceed, `false` to block it.

```csharp
[Middleware("RequireLogin")]
internal class RequireLoginMiddleware : MiddlewareHandler
{
    protected override bool Handle(IRoutable data, SessionTransport session)
    {
        return session is ChatSession { IsAuthenticated: true };
    }
}
```

---

## Dispatch Pipeline

Middleware execution is triggered from `RouteHandler.CanHandle()` for each `[UseMiddleware]` attribute on the route method:

```csharp
RouteHandler.CanHandle(data, session, entry)
    ↓ for each entry.Middlewares[i]
data.Inject(middleware)                         →  attaches the UseMiddlewareAttribute to data
    ↓
MiddlewareHandler.Middleware(data, session)
    ├─ data.GetService<UseMiddlewareAttribute>() →  reads back the injected attribute
    ├─ s_compiledMiddlewares.TryGetValue(name)    →  false if middleware name not registered
    └─ middlewareDelegate(data, session)          →  invokes the compiled Handle()
    ↓
data.Inject<UseMiddlewareAttribute>(null)        →  cleanup, always runs (finally block)
```

If any middleware in the chain returns `false`, `CanHandle()` short-circuits and the route is rejected — the handler method is never invoked.

---

## Startup Registration

At startup (inside `MiddlewareHandler`'s static constructor), for every `[Middleware]`-decorated, non-abstract class:

1. `Lucifer.SetModelI(middleware)` creates the middleware singleton.
2. `RegisterMiddleware(Type t)` reads `[Middleware]` for its `Name`.
3. Looks up the `Handle` method via reflection (`BindingFlags.Instance | NonPublic | Public`).
4. `BuildMiddleware()` compiles a delegate `(IRoutable, SessionTransport) => bool` via `Expression.Lambda`, bound to the singleton instance.
5. Inserts the compiled delegate into `s_compiledMiddlewares` keyed by `classAttr.Name`.

After all classes are registered, `s_compiledMiddlewares.Freeze()` is called once.

---

## Remarks

- Middleware lookup happens by **name string**, not by type — `[UseMiddleware("RequireLogin")]` must exactly match `[Middleware("RequireLogin")]`.
- If a route references a middleware name that was never registered, `Middleware()` returns `false` and the route is blocked by default — fail-closed, not fail-open.
- `data.Inject(...)` / `data.GetService<...>()` is how the currently-executing `UseMiddlewareAttribute` is passed into `Handle()`'s resolution without adding extra parameters to the compiled delegate signature.
- The injected attribute is always cleared in a `finally` block after each middleware call, even if a middleware throws.
- Middleware delegates are compiled once via `Expression.Lambda` and reused for every request — no per-request reflection.

---