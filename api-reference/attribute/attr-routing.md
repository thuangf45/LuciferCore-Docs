# Routing Attributes

**Namespace:** `LuciferCore.Attributes`

Routing attributes map handler methods to incoming messages or HTTP requests. They are evaluated at startup to build an immutable, lock-free dispatch table — no routing overhead occurs at runtime.

There are two categories:

- **`[WsMessage]`** — maps a method to an incoming WebSocket message type.
- **HTTP verb attributes** — map a method to an HTTP method and route path.

All routing attributes inherit from `RouteAttribute`.

---

## `[WsMessage]`

Maps a handler method to an incoming WebSocket binary message by its type name.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class WsMessageAttribute : RouteAttribute
```

### Constructor

```csharp
public WsMessageAttribute(string path)
```

| Parameter | Type | Description |
|---|---|---|
| `path` | `string` | The message type name to match (e.g. `"ChatMessage"`, `"Default"`) |

Internally, `[WsMessage]` uses the method key `"MSG"`. The full dispatch key becomes `/version/prefix/path`.

### Usage

```csharp
[WsMessage("ChatMessage")]
[Safe("")]
[RateLimiter(20, 1)]
[Authorize(UserRole.Guest)]
public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }
```

### Default Handler

Use `"Default"` to catch any message type that does not match a registered handler:

```csharp
[WsMessage("Default")]
[Safe("")]
public void Default([Session] ChatSession session, [Data] PacketModel data)
    => throw new NotImplementedException();
```

---

## HTTP Verb Attributes

Each attribute maps a handler method to a specific HTTP method and an optional sub-route relative to the handler's `[Handler]` prefix.

### Available Attributes

| Attribute | HTTP Method | Declaration |
|---|---|---|
| `[HttpGet(path)]` | `GET` | `public class HttpGetAttribute : RouteAttribute` |
| `[HttpPost(path)]` | `POST` | `public class HttpPostAttribute : RouteAttribute` |
| `[HttpPut(path)]` | `PUT` | `public class HttpPutAttribute : RouteAttribute` |
| `[HttpDelete(path)]` | `DELETE` | `public class HttpDeleteAttribute : RouteAttribute` |
| `[HttpHead(path)]` | `HEAD` | `public class HttpHeadAttribute : RouteAttribute` |
| `[HttpOptions(path)]` | `OPTIONS` | `public class HttpOptionsAttribute : RouteAttribute` |
| `[HttpTrace(path)]` | `TRACE` | `public class HttpTraceAttribute : RouteAttribute` |

All constructors follow the same signature:

```csharp
public Http*Attribute(string path)
```

| Parameter | Type | Description |
|---|---|---|
| `path` | `string` | Sub-route appended to the handler's base prefix. Pass `""` to match the base route exactly |

### Usage

```csharp
[Handler("v1", "/api/user")]
internal class HttpsHandler : HttpsHandlerBase
{
    [HttpGet("")]           // matches GET  /v1/api/user
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }

    [HttpPost("/login")]    // matches POST /v1/api/user/login
    protected void LoginHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }

    [HttpDelete("")]        // matches DELETE /v1/api/user
    protected void DeleteHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
}
```

---

## `RouteAttribute` (Base Class)

All routing attributes inherit from `RouteAttribute`, which normalizes and encodes the method and path into `ByteString` at construction time.

```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public class RouteAttribute : Attribute
```

| Property | Type | Description |
|---|---|---|
| `Method` | `ByteString` | The normalized HTTP or internal method key (e.g. `GET`, `POST`, `MSG`) |
| `Path` | `ByteString` | The normalized route path, always starting with `/` or empty |

`AllowMultiple = true` — a single method can be mapped to multiple routes by stacking routing attributes.

---

## Remarks

- The full dispatch key for any method is assembled as `/{version}/{prefix}/{path}`, normalized at startup.
- Routing tables are built into a lock-free structure once at startup. There is no per-request route matching overhead.
- If no handler matches an incoming message or request, the `"Default"` handler (if registered) is invoked. If no `"Default"` is registered, the message is dropped.
