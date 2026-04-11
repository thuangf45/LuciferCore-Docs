# ResponseModel

**Namespace:** `LuciferCore.Model`

`ResponseModel` is the concrete `HttpModel` subclass for HTTP responses. It maps the three start-line tokens to the standard response layout and adds response-specific builder and parsing methods.

```
Start-line:  HTTP/1.1  SP  200  SP  OK  CRLF
```

| Token | `HttpModel` span | Exposed as |
|---|---|---|
| Protocol | `FirstSpan` | `ProtocolSpan` |
| Status code | `SecondSpan` | *(parsed into `Status` integer on receive)* |
| Status phrase | `ThirdSpan` | `StatusPhraseSpan` |

It also implements `IRoutable` — `MethodRoute` maps to `SecondSpan` (the status code bytes) and `UrlRoute` to `ThirdSpan` (the phrase) — enabling response routing in client-side dispatch scenarios.

`ResponseModel` extends `PooledObject` and implements `IDisposable`. Always obtain instances via `Lucifer.Rent<ResponseModel>()` and release with `using var response = ...` or `Lucifer.Return(response)`.

```csharp
public class ResponseModel : HttpModel, IRoutable
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Status` | `int` | HTTP status code (e.g. `200`, `404`). Populated on receive via `OnHeaderParsed` |
| `StatusPhraseSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the status phrase (e.g. `OK`, `Not Found`) |
| `ProtocolSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the protocol token (e.g. `HTTP/1.1`) |
| `BodySpan` | `ReadOnlySpan<byte>` | Zero-copy span of the response body — inherited from `HttpModel` |
| `Headers` | `int` | Number of headers set — inherited |
| `BodyLength` | `int` | Expected body length as declared by `Content-Length` — inherited |
| `IsEmpty` | `bool` | `true` if the cache buffer is empty — inherited |
| `IsErrorSet` | `bool` | `true` if a parse error occurred during header parsing — inherited |

---

## Quick Response Builders

These methods build a complete, ready-to-send HTTP response in a single call.

### `MakeOkResponse()`

```csharp
public ResponseModel MakeOkResponse(int status = 200)
```

Builds a `200 OK` (or any custom 2xx) response with an empty body.

```csharp
using var response = Lucifer.Rent<ResponseModel>();
session.SendResponse(response.MakeOkResponse());
session.SendResponse(response.MakeOkResponse(204)); // 204 No Content
session.SendResponse(response.MakeOkResponse(201)); // 201 Created
```

### `MakeErrorResponse()`

```csharp
public ResponseModel MakeErrorResponse<TContent>(int status, ReadOnlySpan<TContent> content)
    where TContent : unmanaged

public ResponseModel MakeErrorResponse<TContent, TType>(
    int status,
    ReadOnlySpan<TContent> content,
    ReadOnlySpan<TType> contentType)
    where TContent : unmanaged where TType : unmanaged
```

Builds an error response. Default content type is `text/plain; charset=UTF-8`.

```csharp
using var response = Lucifer.Rent<ResponseModel>();
session.SendResponse(response.MakeErrorResponse(404, "Not found"u8));
session.SendResponse(response.MakeErrorResponse(429, "Too many requests"u8));
session.SendResponse(response.MakeErrorResponse(400, jsonBytes, "application/json"u8));
```

---

## Common Response Builders

| Method | Status | Description |
|---|---|---|
| `MakeHeadResponse()` | `200` | Empty HEAD response |
| `MakeGetResponse(content)` | `200` | GET response with `text/plain` body |
| `MakeGetResponse(content, contentType)` | `200` | GET response with custom content type |
| `MakeOptionsResponse()` | `200` | OPTIONS with default CORS headers (`Allow`, `Access-Control-*`) |
| `MakeOptionsResponse(allow)` | `200` | OPTIONS with custom `Allow` header value |
| `MakeTraceResponse(request)` | `200` | TRACE echo — reflects the `RequestModel` cache as the body |
| `MakeTraceResponse(content)` | `200` | TRACE echo from a raw byte/char span |
| `MakeCustomResponse(status, protocol, content, contentType)` | custom | Full control over status, protocol, body, and content type |

---

## Manual Builder

For full control, chain `SetBegin → SetHeader → SetCookie → SetBody`:

```csharp
using var response = Lucifer.Rent<ResponseModel>();
response
    .SetBegin(200)
    .SetHeader("X-Request-Id"u8, requestId)
    .SetCookie(
        "session"u8, token,
        path: "/"u8,
        maxAge: 3600,
        secure: true,
        strict: true,
        httpOnly: true)
    .SetBody(jsonBytes);

session.SendResponse(response);
```

### `SetBegin(status)`

```csharp
public ResponseModel SetBegin(int status)
```

Writes `HTTP/1.1 {status} {phrase}\r\n` into the cache. The status phrase is resolved automatically from the built-in status phrase table. Calls `Clear()` first.

### `SetBegin(status, protocol)`

```csharp
public ResponseModel SetBegin<TProtocol>(int status, ReadOnlySpan<TProtocol> protocol)
    where TProtocol : unmanaged
```

Same as above but with an explicit protocol token. Pass an empty span to fall back to `HTTP/1.1`.

### `SetHeader(key, value)`

Appends a `key: value\r\n` header line. Inherited from `HttpModel`. Accepts `byte` or `char` spans.

### `SetContentType(extension)`

Sets the `Content-Type` header via the built-in MIME table. Inherited from `HttpModel`.

```csharp
response.SetContentType(".json"u8); // Content-Type: application/json
response.SetContentType(".html"u8); // Content-Type: text/html
```

### `SetCookie(name, value, ...)`

```csharp
public ResponseModel SetCookie<TName, TValue, TPath, TDomain>(
    ReadOnlySpan<TName> name,
    ReadOnlySpan<TValue> value,
    ReadOnlySpan<TPath> path = default,
    ReadOnlySpan<TDomain> domain = default,
    int maxAge = 86400,
    bool secure = true,
    bool strict = true,
    bool httpOnly = true)
```

Appends a `Set-Cookie` header. Each call produces a separate `Set-Cookie` header line.

| Parameter | Default | Description |
|---|---|---|
| `path` | *(empty)* | Cookie path scope |
| `domain` | *(empty)* | Cookie domain scope |
| `maxAge` | `86400` | `Max-Age` in seconds (24 hours) |
| `secure` | `true` | Appends the `Secure` flag |
| `strict` | `true` | Appends `SameSite=Strict` |
| `httpOnly` | `true` | Appends the `HttpOnly` flag |

### `SetBody()` / `SetBody(body)`

Appends the body and writes `Content-Length` automatically. Inherited from `HttpModel`. Call after all `SetHeader` and `SetCookie` calls.

---

## Reading Headers

```csharp
for (var i = 0; i < response.Headers; i++)
{
    if (response.TryGetHeader(i, out var key, out var value))
    {
        // zero-copy ReadOnlySpan<byte> views
    }
}
```

---

## Clone

```csharp
public ResponseModel Clone(bool copyHeaders = true, bool copyCookies = true, bool copyBody = true)
```

Rents a new `ResponseModel` and deep-copies cache and position metadata into it. Useful when you need to hold a response beyond the lifetime of the originating scope.

---

## Implicit Conversion

```csharp
public static implicit operator ReadOnlySpan<byte>(ResponseModel b)
```

Returns a `ReadOnlySpan<byte>` over the entire raw cache. Used by the session's `SendResponse` internals.

---

## Lifecycle & Pooling

```csharp
[HttpGet("")]
protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session)
{
    using var response = Lucifer.Rent<ResponseModel>();
    session.SendResponse(response.MakeGetResponse(jsonBytes, "application/json"u8));
} // Dispose() → Lucifer.Return(response) — buffer returned to pool
```

---

## Remarks

- All span properties are zero-copy views into `Cache`. Do not use them after `Dispose()`.
- `SetBegin` calls `Clear()` unconditionally. A `ResponseModel` can be safely reused within the same scope by calling a builder method, which always starts fresh.
- `Content-Length` is computed and written automatically by `SetBody`. Do not set it manually.
- `Status` is populated only during receive parsing (via `OnHeaderParsed`). When building a response, read the status from the integer you passed to `SetBegin`, not from `Status`.
- `MakeOptionsResponse()` writes `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, `Access-Control-Allow-Credentials`, `Access-Control-Allow-Origin`, and `Access-Control-Max-Age` with default values. Call `SetHeader` after if you need to override specific CORS fields.
- Status phrases are resolved from LuciferCore's built-in phrase table. If you supply an unrecognized status code, the phrase defaults to `Unknown`.
