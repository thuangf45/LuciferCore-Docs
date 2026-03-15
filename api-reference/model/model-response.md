# ResponseModel

**Namespace:** `LuciferCore.Model`

`ResponseModel` is the zero-copy HTTP response builder and parser. It writes status lines, headers, cookies, and body content into a single pooled `Buffer`, serializing everything in place — no intermediate strings, no extra allocations.

It also supports incremental parsing of incoming HTTP responses (client-side use).

It extends `PooledObject` and implements `IDisposable`. Obtain instances via `Lucifer.Rent<ResponseModel>()` and always release with `using var response = ...` or `Lucifer.Return(response)`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Status` | `int` | HTTP status code (e.g. `200`, `404`) |
| `StatusPhraseSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the status phrase (e.g. `OK`, `Not Found`) |
| `ProtocolSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the protocol token (e.g. `HTTP/1.1`) |
| `BodySpan` | `ReadOnlySpan<byte>` | Zero-copy span of the response body |
| `Headers` | `int` | Number of headers set on this response |
| `BodyLength` | `int` | Expected body length as declared by `Content-Length` |
| `IsEmpty` | `bool` | `true` if the internal cache buffer is empty |
| `IsErrorSet` | `bool` | `true` if a parse error occurred during `ReceiveHeader` |
| `Cache` | `Buffer` | The underlying pooled buffer. Lazy-initialized on first access. Both getter and setter call `CheckSafety()` — accessing `Cache` on a disposed `ResponseModel` throws `ObjectDisposedException` |

---

## Quick Response Builders

These methods build a complete, ready-to-send HTTP response in a single call. Use them inside handler methods.

### `MakeOkResponse()`

```csharp
public ResponseModel MakeOkResponse(int status = 200)
```

Builds a `200 OK` (or custom status) response with an empty body.

```csharp
using var response = Lucifer.Rent<ResponseModel>();
session.SendResponse(response.MakeOkResponse());

// Custom status
session.SendResponse(response.MakeOkResponse(204)); // 204 No Content
```

### `MakeErrorResponse()`

```csharp
public ResponseModel MakeErrorResponse<TContent>(int status, ReadOnlySpan<TContent> content)
    where TContent : unmanaged
```

Builds an error response with `text/plain; charset=UTF-8` content type.

```csharp
using var response = Lucifer.Rent<ResponseModel>();
session.SendResponse(response.MakeErrorResponse(404, "Not found"u8));
session.SendResponse(response.MakeErrorResponse(429, "Too many requests"u8));
session.SendResponse(response.MakeErrorResponse(403, "Forbidden"u8));
```

With a custom content type:

```csharp
session.SendResponse(response.MakeErrorResponse(400, jsonBytes, "application/json"u8));
```

---

## Common Response Builders

Pre-assembled responses for standard HTTP scenarios:

| Method | Status | Description |
|---|---|---|
| `MakeHeadResponse()` | `200` | Empty HEAD response |
| `MakeGetResponse(content)` | `200` | GET response with `text/plain` body |
| `MakeGetResponse(content, contentType)` | `200` | GET response with custom content type |
| `MakeOptionsResponse()` | `200` | OPTIONS response with default CORS headers (`Allow`, `Access-Control-*`) |
| `MakeOptionsResponse(allow)` | `200` | OPTIONS with custom `Allow` header |
| `MakeTraceResponse(request)` | `200` | TRACE echo — reflects the `RequestModel` cache as body |
| `MakeCustomResponse(status, protocol, content, contentType)` | custom | Full custom response with any status, protocol, body, and type |

---

## Manual Builder

For full control, use the fluent `SetBegin → SetHeader → SetCookie → SetBody` chain:

```csharp
using var response = Lucifer.Rent<ResponseModel>();
response
    .SetBegin(200)
    .SetHeader("X-Request-Id"u8, requestId)
    .SetCookie("session"u8, token, path: "/"u8, maxAge: 3600, secure: true, strict: true, httpOnly: true)
    .SetBody(jsonBytes);

session.SendResponse(response);
```

### `SetBegin(status)`

```csharp
public ResponseModel SetBegin(int status)
```

Sets the HTTP/1.1 status line. Status phrase is resolved automatically from the status code. Clears the response before writing.

### `SetHeader(key, value)`

```csharp
public ResponseModel SetHeader<TKey, TValue>(ReadOnlySpan<TKey> key, ReadOnlySpan<TValue> value)
    where TKey : unmanaged where TValue : unmanaged
```

Appends a response header. Accepts `byte` (UTF-8) or `char` spans for both key and value.

### `SetContentType(extension)`

```csharp
public ResponseModel SetContentType<TExt>(ReadOnlySpan<TExt> extension)
    where TExt : unmanaged
```

Sets the `Content-Type` header by looking up the file extension in LuciferCore's built-in MIME table. No-ops if the extension is not found.

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

Appends a `Set-Cookie` header with optional attributes:

| Parameter | Default | Description |
|---|---|---|
| `path` | *(empty)* | Cookie path scope |
| `domain` | *(empty)* | Cookie domain scope |
| `maxAge` | `86400` | Max-Age in seconds (default 24 hours) |
| `secure` | `true` | Adds `Secure` flag |
| `strict` | `true` | Adds `SameSite=Strict` flag |
| `httpOnly` | `true` | Adds `HttpOnly` flag |

### `SetBody(body)`

```csharp
public ResponseModel SetBody<T>(ReadOnlySpan<T> body) where T : unmanaged
```

Appends the response body and sets `Content-Length` automatically. Handles UTF-8 byte counting when `T` is `char`. Call after all `SetHeader` calls.

### `SetBody()`

Appends an empty body with `Content-Length: 0`. Use for responses that require a body-terminating CRLF (e.g. `200 OK` with no content).

---

## Reading Headers

```csharp
public bool TryGetHeader(int i, out ReadOnlySpan<byte> key, out ReadOnlySpan<byte> value)
```

Retrieves the header at index `i` as zero-copy spans. Returns `false` if the index is out of range.

---

## Implicit Conversion

```csharp
public static implicit operator ReadOnlySpan<byte>(ResponseModel b)
```

Implicitly converts the response to a `ReadOnlySpan<byte>` over the full raw cache. Used by the session's `SendResponse` internals.

---

## Lifecycle & Pooling

Always rent and return `ResponseModel` within the same handler scope:

```csharp
[HttpGet("")]
protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session)
{
    using var response = Lucifer.Rent<ResponseModel>();
    session.SendResponse(response.MakeGetResponse(jsonBytes, "application/json"u8));
}
```

The `using` statement calls `Dispose()`, which calls `Lucifer.Return(this)`, returning the instance and its internal `Buffer` to their respective pools.

---

## Remarks

- All span properties are zero-copy views into the `Cache` buffer. Do not use them after `Dispose()`.
- `SetBegin()` always calls `Clear()` internally — you cannot accidentally append to a previous response.
- `Content-Length` is calculated and written automatically by `SetBody()`. Do not set it manually.
- The MIME table for `SetContentType()` is built into `StorageData` and covers all common web file types.
- `MakeOptionsResponse()` sets `Access-Control-Allow-Methods`, `Access-Control-Allow-Headers`, and `Access-Control-Allow-Credentials` automatically. Override with `SetHeader()` calls after if you need custom CORS values.
