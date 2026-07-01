# RequestModel

**Namespace:** `LuciferCore.Model`

`RequestModel` is the HTTP request model in LuciferCore.

It maps start-line tokens like this:

```text
METHOD SP URL SP HTTP/1.1 CRLF
```

| Token | Base span | RequestModel name |
|---|---|---|
| Method | `FirstSpan` | `MethodSpan` |
| URL | `SecondSpan` | `UrlSpan` |
| Protocol | `ThirdSpan` | `ProtocolSpan` |

It also implements `IRoutable` for dispatch.

```csharp
public class RequestModel : HttpModel<RequestModel>, IRoutable
```

---

## Main properties

| Property | Type | Meaning |
|---|---|---|
| `MethodSpan` | `ReadOnlySpan<byte>` | HTTP method (`GET`, `POST`, ...) |
| `UrlSpan` | `ReadOnlySpan<byte>` | Full URL (can include query) |
| `ProtocolSpan` | `ReadOnlySpan<byte>` | Protocol text (`HTTP/1.1`) |
| `BodySpan` | `ReadOnlySpan<byte>` | Request body |
| `Headers` | `int` | Header count |
| `Cookies` | `int` | Cookie count |
| `BodyLength` | `int` | Body length |
| `IsEmpty` | `bool` | Buffer empty state |
| `IsErrorSet` | `bool` | Parse error flag |

---

## Read incoming request

### Read headers

```csharp
for (var i = 0; i < request.Headers; i++)
{
    if (request.TryGetHeader(i, out var key, out var value))
    {
        // use key/value span
    }
}
```

### Read cookies

```csharp
for (var i = 0; i < request.Cookies; i++)
{
    if (request.TryGetCookie(i, out var name, out var value))
    {
        // use cookie
    }
}
```

### Read body

```csharp
var body = request.BodySpan;
```

---

## Build outgoing request (client side)

### Quick builders

```csharp
using var req = Lucifer.Rent<RequestModel>();

req.MakeGetRequest("/api/users"u8);
req.MakeHeadRequest("/api/health"u8);
req.MakeDeleteRequest("/api/user/1"u8);

req.MakePostRequest("/api/login"u8, jsonBytes);
req.MakePostRequest("/api/login"u8, jsonBytes, "application/json"u8);

req.MakePutRequest("/api/user/1"u8, jsonBytes);
req.MakePutRequest("/api/user/1"u8, jsonBytes, "application/json"u8);

req.MakeCustomRequest("PATCH"u8, "/api/user/1"u8, jsonBytes, "application/json"u8);
```

### Manual builder

```csharp
using var req = Lucifer.Rent<RequestModel>();
req.SetBegin("POST"u8, "/api/login"u8)
   .SetHeader("Content-Type"u8, "application/json"u8)
   .SetHeader("X-Request-Id"u8, requestId)
   .SetCookie("session"u8, sessionToken)
   .AddCookie("lang"u8, "en"u8)
   .SetBody(jsonBytes);
```

---

## Cookie helpers

```csharp
req.SetCookie("session"u8, "abc123"u8);
req.AddCookie("lang"u8, "en"u8);
```

Result:

```text
Cookie: session=abc123; lang=en
```

---

## Clone

`RequestModel` uses inherited CRTP clone:

```csharp
public RequestModel Clone(bool copyHeaders = true, bool copyCookies = true, bool copyBody = true)
```

Use clone when processing in background task.

---

## Implicit conversion

```csharp
public static implicit operator ReadOnlySpan<byte>(RequestModel b)
```

You can pass request as raw bytes directly when needed.

---

## Pooling and lifecycle

In handler methods, request is managed by framework.  
Do not dispose it manually.

```csharp
[HttpPost("")]
protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session)
{
    // framework owns request
}
```

Only dispose models that you rent/clone yourself.

---

## Notes

- Inherit from `HttpModel<RequestModel>` (CRTP) to keep fluent API.
- `UrlSpan` may include query string.
- Routing layer can strip query during route matching.
- All spans are zero-copy views of pooled buffer.
- Do not keep spans after model is returned/disposed.
- `ToString()` is for debug only (allocates string).