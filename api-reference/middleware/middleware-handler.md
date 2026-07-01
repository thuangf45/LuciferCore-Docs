# MiddlewareHandler

**Namespace:** `LuciferCore.Handler`

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
private static readonly Utf8Map<Func<IRoutable, SessionTransport, bool>> s_compiledMiddlewares;
```

This map stores:

`middleware name -> compiled Handle delegate`

It is built once at startup, then frozen for fast reads.

---

## `Handle(...)`

```csharp
protected abstract bool Handle(IRoutable data, SessionTransport session);
```

Every middleware class must implement this method.

- return `true` => continue
- return `false` => block route

Example:

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
4. Compiles delegate `(IRoutable, SessionTransport) => bool`
5. Stores delegate in registry using middleware name
6. Freezes registry

No per-request reflection.

---

## Notes

- Middleware resolution is by **name**:
  - `[UseMiddleware("RequireLogin")]`
  - must match `[Middleware("RequireLogin")]`
- If middleware name is missing/not registered, route is blocked (fail-closed).
- You can chain many middlewares with `Order`.
- Middleware should stay lightweight and fast.