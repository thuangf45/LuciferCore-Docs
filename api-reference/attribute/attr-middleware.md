# Middleware Attributes

**Namespace:** `LuciferCore.Attributes`

Middleware attributes add shared logic before handler code runs.

Common use cases:
- auth check
- rate limit
- session check
- custom security rules

Main attributes:
- `[Middleware]` (declare middleware class)
- `[UseMiddleware]` (use middleware on method)
- `[Authorize]` (built-in role check)
- `[RateLimiter]` (built-in rate limit)

---

## `[Middleware]`

Marks a class as middleware.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class MiddlewareAttribute : Attribute
```

### Constructor

```csharp
public MiddlewareAttribute(string name)
```

| Parameter | Type | Meaning |
|---|---|---|
| `name` | `string` | Middleware name |

### Properties

| Property | Type | Meaning |
|---|---|---|
| `Name` | `ByteString` | UTF-8 middleware name |

### Usage

```csharp
[Middleware("Session")]
public class SessionMiddleware : MiddlewareHandler
{
    protected override bool Handle(IRoutable data, SessionTransport session) => true;
}
```

---

## `[UseMiddleware]`

Adds middleware to a handler method.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public class UseMiddlewareAttribute : Attribute
```

### Constructor

```csharp
public UseMiddlewareAttribute(string name)
```

| Parameter | Type | Meaning |
|---|---|---|
| `name` | `string` | Middleware name to run |

### Properties

| Property | Type | Meaning |
|---|---|---|
| `Name` | `ByteString` | Middleware name |
| `Order` | `int` | Run order (default `0`) |

### Usage

```csharp
[Message("ChatMessage")]
[UseMiddleware("Logging", Order = 1)]
public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }
```

- You can stack many middleware attributes.
- Lower `Order` runs first.

---

## `[Authorize]`

Built-in role check.

`[Authorize]` derives from `UseMiddlewareAttribute` and uses middleware name `"Session"`.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class AuthorizeAttribute : UseMiddlewareAttribute
```

### Constructor

```csharp
public AuthorizeAttribute(UserRole minRole = UserRole.User)
```

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `minRole` | `UserRole` | `UserRole.User` | Minimum role required |

### Properties

| Property | Type | Meaning |
|---|---|---|
| `MinRole` | `UserRole` | Required minimum role |
| `Name` | `ByteString` | Always `"Session"` |
| `Order` | `int` | Middleware order |

### Usage

```csharp
[Message("ChatMessage")]
[Authorize(UserRole.Guest)]
public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }
```

Notes:
- Method-level only. Cannot be applied to a class.
- If no `[Authorize]`, no role check.

---

## `[RateLimiter]`

Built-in rate limiter.

`[RateLimiter]` derives from `UseMiddlewareAttribute` and uses middleware name `"RateLimit"`.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Class)]
public class RateLimiterAttribute : UseMiddlewareAttribute
```

### Constructor

```csharp
public RateLimiterAttribute(int limit = 1, int periodSeconds = 60)
```

| Parameter | Type | Default | Meaning |
|---|---|---|---|
| `limit` | `int` | `1` | Max requests in window |
| `periodSeconds` | `int` | `60` | Window size (seconds) |

### Properties

| Property | Type | Meaning |
|---|---|---|
| `Limit` | `int` | Max requests |
| `PeriodSeconds` | `int` | Time window |
| `Name` | `ByteString` | Always `"RateLimit"` |
| `Order` | `int` | Middleware order |

### Usage

```csharp
[RateLimiter(10, 1)]
public partial class ChatSession : WssSession { ... }
```

```csharp
[Handler("v1", "/api/user")]
[RateLimiter(100, 1)]
internal class UserHandler : RouteHandler { ... }
```

Notes:
- Class-level only.
- Cannot be applied to a single method.
- Applies to the whole class.

---

## Extend middleware system

Middleware is open for extension:

- Create your own middleware class with `[Middleware("Name")]`
- Apply it by `[UseMiddleware("Name")]`
- Stack many middlewares in one route
- Control order with `Order`

See:
- **API Reference** for built-in middleware
- **Middleware Guide** for custom middleware design