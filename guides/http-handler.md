# HTTP Handler

An **HTTP Handler** processes incoming HTTPS requests dispatched by `Lucifer.Dispatch()`. Each method is mapped to an HTTP verb and route via the corresponding verb attribute.

---

## The `[Handler]` Attribute

```csharp
[Handler("v1", "/api/user")]
internal class HttpsHandler : HttpsHandlerBase { ... }
```

| Parameter | Type | Description |
|---|---|---|
| `version` | `string` | API version namespace (e.g. `"v1"`) |
| `route` | `string` | Base route for this handler (e.g. `"/api/user"`) |

---

## HTTP Verb Attributes

Override the base class methods and decorate each with the appropriate verb attribute:

| Attribute | HTTP Method | Override Method |
|---|---|---|
| `[HttpGet("")]` | `GET` | `GetHandle` |
| `[HttpPost("")]` | `POST` | `PostHandle` |
| `[HttpPut("")]` | `PUT` | `PutHandle` |
| `[HttpDelete("")]` | `DELETE` | `DeleteHandle` |
| `[HttpHead("")]` | `HEAD` | `HeadHandle` |
| `[HttpOptions("")]` | `OPTIONS` | `OptionsHandle` |
| `[HttpTrace("")]` | `TRACE` | `TraceHandle` |

The string parameter appended to each attribute is a sub-route relative to the handler's base route.

---

## Method Parameters

| Attribute | Type | Description |
|---|---|---|
| `[Data]` | `RequestModel` | The incoming HTTP request |
| `[Session]` | `HttpsSession` | The active client session |

---

## Security Attributes

The same security attributes available on WebSocket handlers apply here:

| Attribute | Description |
|---|---|
| `[Safe("")]` | Wraps execution in a safe context |
| `[Authorize(role)]` | Enforces role-based access |
| `[RateLimiter(limit, window)]` | Per-method rate limiting |

---

## Full Example

```csharp
[Handler("v1", "/api/user")]
internal class HttpsHandler : HttpsHandlerBase
{
    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpHead("")]
    protected void HeadHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpGet("")]
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpPost("")]
    protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpPut("")]
    protected void PutHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpDelete("")]
    protected void DeleteHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpTrace("")]
    protected void TraceHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [Safe("")][Authorize(UserRole.Guest)][RateLimiter(100, 1)][HttpOptions("")]
    protected void OptionsHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();
}
```
