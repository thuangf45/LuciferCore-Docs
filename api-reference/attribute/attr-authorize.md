# [Authorize]

**Namespace:** `LuciferCore.Attributes`

Enforces role-based access control on a handler method or class. The session's current `UserRole` is checked against the declared minimum role before the handler body executes. If the session does not meet the minimum role, the request is rejected.

---

## Declaration

```csharp
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class, Inherited = true, AllowMultiple = false)]
public class AuthorizeAttribute : Attribute
```

---

## Constructor

```csharp
public AuthorizeAttribute(UserRole minRole = UserRole.User)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `minRole` | `UserRole` | `UserRole.User` | Minimum role required to access this method or class |

---

## Properties

| Property | Type | Description |
|---|---|---|
| `MinRole` | `UserRole` | The minimum `UserRole` required for access |

---

## UserRole Enum

`UserRole` is defined in `LuciferCore.Manager`. Roles are ordered by privilege level:

| Value | Description |
|---|---|
| `UserRole.Guest` | Unauthenticated or anonymous session — lowest privilege |
| `UserRole.User` | Authenticated standard user (default minimum) |
| *(additional roles)* | Defined by your application's role hierarchy |

---

## Usage

### Method-level

Restrict a single handler method to a specific minimum role:

```csharp
[WsMessage("ChatMessage")]
[Authorize(UserRole.Guest)] // accessible to all sessions including unauthenticated
public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }
```

```csharp
[HttpDelete("")]
[Authorize(UserRole.User)] // requires an authenticated user
protected void DeleteHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
```

### Class-level

Apply a default role requirement to all methods in a handler class. Individual methods can override with their own `[Authorize]`:

```csharp
[Handler("v1", "/api/admin")]
[Authorize(UserRole.User)]
internal class AdminHandler : HttpsHandlerBase { ... }
```

---

## Remarks

- `Inherited = true` — role requirements are inherited by subclasses and overriding methods unless redeclared.
- `AllowMultiple = false` — only one `[Authorize]` per target.
- Authorization is evaluated in the dispatch pipeline **after** `[RateLimiter]` and **before** `[Safe]`.
- If no `[Authorize]` is declared, no role check is performed and the method is accessible to all sessions.
