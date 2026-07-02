# MiddlewareHandler

**Namespace:** `LuciferCore.Middleware`

`MiddlewareHandler` is the base class for all middleware.

It manages:
- middleware registry
- middleware lookup by name
- compiled middleware delegates

```csharp
public abstract class MiddlewareHandler
```

---

## Shared registry

```csharp
private static readonly Utf8Map<Func<IRoutable?, SessionTransport, bool>> s_compiledMiddlewares;
```

This map stores:

`middleware name -> compiled Handle delegate`

It is built once at startup, then frozen for fast reads.

---

## Handle()

```csharp
protected abstract bool Handle(IRoutable? data, SessionTransport session);
```

Every middleware class must implement this method.

- Return `true` to continue.
- Return `false` to block the route.

**Note:**  
- `data` can be `null`.   
- Session-level middleware runs with `data = null`.  
- Route-level middleware runs with real `data`.  
- Always check for `null` before using it.

Example:

```csharp
[Middleware("RequireLogin")]
internal class RequireLoginMiddleware : MiddlewareHandler
{
    protected override bool Handle(IRoutable? data, SessionTransport session)
    {
        return session is ChatSession { IsAuthenticated: true };
    }
}
```

---

## Runtime flow

Middleware runs before handler method.

Simple flow:

```text
RouteHandler checks route middlewares
 -> get middleware name from [UseMiddleware]
 -> find middleware delegate in registry
 -> execute Handle(data, session)
 -> if false: stop route
 -> if true: continue to next middleware/handler
```

If any middleware fails, handler method is not called.

---

## Startup registration

At startup, LuciferCore:

1. Finds all classes with `[Middleware]`
2. Creates middleware singleton instance
3. Finds `Handle(...)` method
4. Compiles delegate `(IRoutable?, SessionTransport) => bool`
5. Stores delegate in registry using middleware name
6. Freezes registry

No per-request reflection.

---

## Notes

- Middleware resolution is by **name**:
  - `[UseMiddleware("RequireLogin")]`
  - must match `[Middleware("RequireLogin")]`
- If there is no `[UseMiddleware]`, the route runs normally.
- If `[UseMiddleware]` is set but the name is not in the registry, the route is blocked (fail-closed).
- You can chain many middlewares with `Order`.
- Middleware should stay lightweight and fast.