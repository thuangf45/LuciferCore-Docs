# ResponseModel

**Namespace:** `LuciferCore.Model`

`ResponseModel` is the HTTP response model in LuciferCore.

Start-line format:

```text
HTTP/1.1 SP 200 SP OK CRLF
```

| Token | Base span | ResponseModel name |
|---|---|---|
| Protocol | `FirstSpan` | `ProtocolSpan` |
| Status code | `SecondSpan` | `Status` (parsed/set as int) |
| Status phrase | `ThirdSpan` | `StatusPhraseSpan` |

`ResponseModel` also implements `IRoutable`.

```csharp
public class ResponseModel : HttpModel<ResponseModel>, IRoutable
```

---

## Main properties

| Property | Type | Meaning |
|---|---|---|
| `Status` | `int` | HTTP status code (`200`, `404`, ...) |
| `ProtocolSpan` | `ReadOnlySpan<byte>` | Protocol text (`HTTP/1.1`) |
| `StatusPhraseSpan` | `ReadOnlySpan<byte>` | Status phrase (`OK`, `Not Found`) |
| `BodySpan` | `ReadOnlySpan<byte>` | Response body |
| `Headers` | `int` | Header count |
| `Cookies` | `int` | Parsed cookie count |
| `BodyLength` | `int` | Body length |
| `IsEmpty` | `bool` | Buffer empty state |
| `IsErrorSet` | `bool` | Parse error flag |

---

## Read incoming response (client side)

### Headers

```csharp
for (var i = 0; i < response.Headers; i++)
{
    if (response.TryGetHeader(i, out var key, out var value))
    {
        // use header key/value
    }
}
```

### Cookies

```csharp
for (var i = 0; i < response.Cookies; i++)
{
    if (response.TryGetCookie(i, out var name, out var value))
    {
        // parsed from Set-Cookie
    }
}
```

### Body

```csharp
var body = response.BodySpan;
```

---

## Quick builders

```csharp
using var response = Lucifer.Rent<ResponseModel>();

response.MakeOkResponse();                    // 200
response.MakeOkResponse(204);                 // 204 No Content
response.MakeErrorResponse(404, "Not found"u8);
response.MakeErrorResponse(400, jsonBytes, "application/json"u8);
```

---

## Common builders

| Method | Meaning |
|---|---|
| `MakeHeadResponse()` | Build HEAD response |
| `MakeGetResponse(content)` | 200 with body |
| `MakeGetResponse(content, contentType)` | 200 with custom content type |
| `MakeOptionsResponse()` | OPTIONS with default CORS headers |
| `MakeOptionsResponse(allow)` | OPTIONS with custom Allow |
| `MakeTraceResponse(request)` | TRACE echo from request |
| `MakeTraceResponse(content)` | TRACE echo from raw content |
| `MakeCustomResponse(...)` | Full custom response |

---

## Manual builder

```csharp
using var response = Lucifer.Rent<ResponseModel>();
response
    .Clear()
    .SetBegin(200)
    .SetHeader("X-Request-Id"u8, requestId)
    .SetCookie("session"u8, token, path: "/"u8, maxAge: 3600, secure: true, strict: true, httpOnly: true)
    .SetBody(jsonBytes);

session.SendResponse(response);
```

---

## `SetBegin(...)` overloads

| Method | Meaning |
|---|---|
| `SetBegin(status)` | Use status with default protocol |
| `SetBegin(status, protocol)` | Use custom protocol |
| `SetBegin(status, statusPhrase, protocol)` | Full custom start-line |

---

## Cookie builder

`SetCookie(...)` always writes one `Set-Cookie` header line per call.

Call it multiple times for multiple cookies.

---

## Clone

Inherited CRTP clone returns `ResponseModel`:

```csharp
public ResponseModel Clone(bool copyHeaders = true, bool copyCookies = true, bool copyBody = true)
```

---

## Implicit conversion

```csharp
public static implicit operator ReadOnlySpan<byte>(ResponseModel b)
```

Can send as raw bytes directly.

---

## Pooling lifecycle

```csharp
[HttpGet("")]
protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session)
{
    using var response = Lucifer.Rent<ResponseModel>();
    session.SendResponse(response.MakeGetResponse(jsonBytes, "application/json"u8));
}
```

---

## Notes

- Use `HttpModel<ResponseModel>` (CRTP) to keep full fluent API.
- Span properties are zero-copy views into pooled buffer.
- Do not use spans after dispose/return.
- `SetBody(...)` writes `Content-Length` automatically.
- Quick builders call `Clear()` internally.
- If manual flow reuses instance, call `Clear()` first.
- `ToString()` is for debug only.