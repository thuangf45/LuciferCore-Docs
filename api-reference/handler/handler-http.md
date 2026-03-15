# HTTP Handler Bases

**Namespace:** `LuciferCore.Handler.Transports`

LuciferCore provides two HTTP handler base classes depending on whether the connection is encrypted:

| Class | Protocol | Session Type |
|---|---|---|
| `HttpHandlerBase` | HTTP (unencrypted) | `HttpSession` |
| `HttpsHandlerBase` | HTTPS (TLS) | `HttpsSession` |

Both classes share identical behavior and API surface. The only difference is the session type they bind to and the internal response method (`SendResponseAsync` vs `SendResponse`).

---

## HttpsHandlerBase

```csharp
public abstract class HttpsHandlerBase : HandlerBase, IHandler<RequestModel, HttpsSession>
```

Base class for all HTTPS handlers. Extend this when your server is declared as `WssServer` (which handles both WSS and HTTPS on the same port).

### HttpHandlerBase

```csharp
public abstract class HttpHandlerBase : HandlerBase, IHandler<RequestModel, HttpSession>
```

Base class for unencrypted HTTP handlers. Extend this when your server is declared as `HttpServer`.

---

## Route Key Format

Both bases build the dispatch key from `RequestModel.MethodSpan` and `RequestModel.UrlSpan`. Query strings are automatically stripped:

```
{HTTP-METHOD}:/{version}/{prefix}/{sub-route}

// Examples:
GET:/v1/api/user
POST:/v1/api/user/login
DELETE:/v1/api/user
OPTIONS:/v1/api/user
```

The query string (everything after `?`) is excluded from the key automatically. Route parameters are not yet parsed at this stage.

---

## Extending

Decorate your class with `[Handler]` and declare one method per HTTP verb you want to handle. There is no requirement to implement all verbs.

```csharp
[Handler("v1", "/api/user")]
internal class HttpsHandler : HttpsHandlerBase
{
    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpGet("")]
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session)
    {
        // handle GET /v1/api/user
    }

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpPost("")]
    protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session)
    {
        // handle POST /v1/api/user
    }

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpDelete("")]
    protected void DeleteHandle([Data] RequestModel request, [Session] HttpsSession session)
    {
        // handle DELETE /v1/api/user
    }
}
```

### Sub-routes

Pass a path string to the verb attribute to register a sub-route relative to the handler's base prefix:

```csharp
[Handler("v1", "/api/user")]
internal class HttpsHandler : HttpsHandlerBase
{
    [HttpGet("")]           // GET  /v1/api/user
    protected void GetAll([Data] RequestModel req, [Session] HttpsSession s) { ... }

    [HttpGet("/profile")]   // GET  /v1/api/user/profile
    protected void GetProfile([Data] RequestModel req, [Session] HttpsSession s) { ... }

    [HttpPost("/login")]    // POST /v1/api/user/login
    protected void Login([Data] RequestModel req, [Session] HttpsSession s) { ... }
}
```

---

## Error & OK Responses

Both bases override `Error()` and `Ok()` from `HandlerBase` to send proper HTTP responses via `ResponseModel`:

| Method | When triggered | HTTP behavior |
|---|---|---|
| `Error(session, 404, ...)` | Route not found | Sends HTTP 404 response |
| `Error(session, 429, ...)` | Rate limit exceeded | Sends HTTP 429 response |
| `Error(session, 403, ...)` | Insufficient role | Sends HTTP 403 response |
| `Ok(session, 200)` | Explicit success | Sends HTTP 200 response |

You can also call `Ok()` and `Error()` manually inside your handler methods to send explicit responses.

---

## Supported HTTP Verbs

All standard HTTP methods are supported via routing attributes. Each maps to a method name by convention, though you can name your methods freely:

| Attribute | HTTP Method | Conventional method name |
|---|---|---|
| `[HttpGet(path)]` | `GET` | `GetHandle` |
| `[HttpPost(path)]` | `POST` | `PostHandle` |
| `[HttpPut(path)]` | `PUT` | `PutHandle` |
| `[HttpDelete(path)]` | `DELETE` | `DeleteHandle` |
| `[HttpHead(path)]` | `HEAD` | `HeadHandle` |
| `[HttpOptions(path)]` | `OPTIONS` | `OptionsHandle` |
| `[HttpTrace(path)]` | `TRACE` | `TraceHandle` |

---

## Multiple Handlers on the Same Base Route

You can split handlers by concern across multiple classes sharing the same version, with different route prefixes:

```csharp
[Handler("v1", "/api/user")]
internal class UserHandler : HttpsHandlerBase { ... }

[Handler("v1", "/api/admin")]
internal class AdminHandler : HttpsHandlerBase { ... }

[Handler("v1", "/api/product")]
internal class ProductHandler : HttpsHandlerBase { ... }
```

Each class gets its own prefix and its own set of registered routes in the shared routing table.

---

## Remarks

- Do not call `Handle()` directly — it is invoked by `Lucifer.Dispatch()` through the pipeline.
- `Handle()` is `virtual` — override it to intercept all dispatch for a handler before route resolution.
- `HttpsHandlerBase` uses `SendResponse()` (synchronous) while `HttpHandlerBase` uses `SendResponseAsync()`. This matches the underlying `NetCoreServer` session contract for each transport type.
- Unlike WebSocket handlers, HTTP handlers have no built-in default handler. Unmatched routes return HTTP 404 automatically via `Error(session, 404, ...)`.
- The query string is stripped automatically from the route key. Access query parameters through `RequestModel` inside your handler method.
