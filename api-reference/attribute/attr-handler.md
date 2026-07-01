# [Handler]

**Namespace:** `LuciferCore.Attributes`

Marks a class as a route handler.

LuciferCore auto-discovers all `[Handler]` classes at startup and registers their routes.

> Handler class should inherit `RouteHandler` (single handler base).  
> Older protocol-specific handler base classes are no longer required.

---

## Declaration

```csharp
[AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
public sealed class HandlerAttribute : Attribute
```

---

## Constructor

```csharp
public HandlerAttribute(string version = "v1", string prefix = "")
```

| Parameter | Type | Meaning |
|---|---|---|
| `version` | `string` | API version (default: `"v1"`) |
| `prefix` | `string` | Base route/scope for this handler |

---

## Properties

| Property | Type | Meaning |
|---|---|---|
| `Version` | `ByteString` | UTF-8 version (example: `/v1`) |
| `Prefix` | `ByteString` | UTF-8 prefix (example: `/api/user`, `/wss`) |

---

## Usage

### WebSocket-style scope

```csharp
[Handler("v1", "wss")]
internal class ChatHandler : RouteHandler
{
    [Message("ChatMessage")]
    public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }
}
```

### HTTP-style scope

```csharp
[Handler("v1", "/api/user")]
internal class UserHandler : RouteHandler
{
    [HttpGet("")]
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
}
```

---

## Route resolution

Final route key is built from:

`version + prefix + method-level route`

Examples:

```text
/v1/api/user + [HttpGet("")]         -> GET /v1/api/user
/v1/api/user + [HttpPost("login")]   -> POST /v1/api/user/login
/v1/wss      + [Message("Chat")]     -> MSG /v1/wss/Chat
```

---

## Notes

- One `[Handler]` per class (`AllowMultiple = false`).
- Routing table is built at startup (no per-request reflection).
- Do not instantiate handlers manually.
- Protocol behavior comes from method attributes (`[Message]`, `[HttpGet]`, ...) and `[Data]` type, not from different handler base classes.
- You can extend routing with custom route labels/attributes for custom protocols.