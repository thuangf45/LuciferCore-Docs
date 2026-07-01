# Routing Attributes

**Namespace:** `LuciferCore.Attributes`

Routing attributes map handler methods to incoming data.

LuciferCore reads these attributes at startup and builds a fixed routing table.

Two main groups:

- `[Message]` for message-style routes (for example WS/WSS/custom message protocols)
- HTTP verb attributes (`[HttpGet]`, `[HttpPost]`, ...)

All of them inherit from `RouteAttribute`.

---

## `[Message]`

Maps a method to a message name.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class MessageAttribute : RouteAttribute
```

`[Message]` allows one route per method (`AllowMultiple = false`).

### Constructor

```csharp
public MessageAttribute(string path)
```

| Parameter | Type | Meaning |
|---|---|---|
| `path` | `string` | Message name (example: `"ChatMessage"`, `"Default"`) |

Internal method key is `"MSG"`.

### Usage

```csharp
[Message("ChatMessage")]
[Authorize(UserRole.Guest)]
public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }
```

### Default message route

```csharp
[Message("Default")]
public void Default([Session] ChatSession session, [Data] PacketModel data)
    => throw new NotImplementedException();
```

Use `"Default"` as fallback when no message route matches.

---

## HTTP verb attributes

Map method to HTTP verb + sub-route.

### Available attributes

| Attribute | HTTP Method |
|---|---|
| `[HttpGet(path)]` | GET |
| `[HttpPost(path)]` | POST |
| `[HttpPut(path)]` | PUT |
| `[HttpDelete(path)]` | DELETE |
| `[HttpHead(path)]` | HEAD |
| `[HttpOptions(path)]` | OPTIONS |
| `[HttpTrace(path)]` | TRACE |

These attributes inherit `AllowMultiple = true` from `RouteAttribute`, so you can stack them.

Constructor form:

```csharp
public Http*Attribute(string path)
```

| Parameter | Type | Meaning |
|---|---|---|
| `path` | `string` | Sub-route under handler prefix (`""` = exact base route) |

### Usage

```csharp
[Handler("v1", "/api/user")]
internal class UserHandler : RouteHandler
{
    [HttpGet("")]
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }

    [HttpPost("login")]
    protected void LoginHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }

    [HttpDelete("")]
    protected void DeleteHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
}
```

---

## `RouteAttribute` (base class)

```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public class RouteAttribute : Attribute
```

| Property | Type | Meaning |
|---|---|---|
| `Method` | `ByteString` | Method key (`GET`, `POST`, `MSG`, ...) |
| `Path` | `ByteString` | Normalized path |

### Normalization rules

- If method is empty, method becomes `"ANY"`.
- If path is empty, path is empty.
- If path does not start with `/`, `/` is added automatically.

---

## Route key format

Final route key is built from:

`/{version}/{prefix}/{path}`

Example:

```text
/v1/api/user + [HttpGet("")]       -> GET /v1/api/user
/v1/api/user + [HttpPost("login")] -> POST /v1/api/user/login
/v1/wss      + [Message("Chat")]   -> MSG /v1/wss/Chat
```

---

## Notes

- Routing is prepared once at startup.
- Runtime dispatch uses the prebuilt table.
- If no route matches, `"Default"` route is used (if defined).
- Routing is extensible: you can create custom route labels/attributes for custom protocols.