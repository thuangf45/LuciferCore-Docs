# RequestModel

**Namespace:** `LuciferCore.Model`

`RequestModel` is the zero-copy container for incoming HTTP requests. It parses and stores the request start-line, headers, cookies, and body as position metadata into a single pooled `Buffer` — every read is a zero-allocation span view into that buffer.

It also serves as a builder for outgoing HTTP requests (client-side use). All builder methods are fluent and return `this`.

It extends `PooledObject` and implements `IDisposable`. Always obtain instances via the session's `OnReceivedRequest` callback — the framework manages the lifecycle automatically. If you clone a request, release the clone with `Lucifer.Return(clone)`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `MethodSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the HTTP method (e.g. `GET`, `POST`) |
| `UrlSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the request URL including path and query string |
| `ProtocolSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the protocol version (e.g. `HTTP/1.1`) |
| `BodySpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8/raw span of the request body |
| `Headers` | `int` | Number of parsed headers |
| `Cookies` | `int` | Number of parsed cookies |
| `BodyLength` | `int` | Expected body length in bytes as declared by `Content-Length` |
| `IsEmpty` | `bool` | `true` if the internal cache buffer is empty |
| `IsErrorSet` | `bool` | `true` if a parse error occurred during `ReceiveHeader` |
| `Cache` | `Buffer` | The underlying pooled buffer. Lazy-initialized on first access. Both getter and setter call `CheckSafety()` — accessing `Cache` on a disposed `RequestModel` throws `ObjectDisposedException` |

---

## Reading Data

### Headers

```csharp
public bool TryGetHeader(int i, out ReadOnlySpan<byte> key, out ReadOnlySpan<byte> value)
```

Retrieves the header at index `i` as zero-copy spans. Returns `false` if the index is out of range.

```csharp
for (var i = 0; i < request.Headers; i++)
{
    if (request.TryGetHeader(i, out var key, out var value))
    {
        // key and value are ReadOnlySpan<byte> views into the buffer
    }
}
```

### Cookies

```csharp
public bool TryGetCookie(int i, out ReadOnlySpan<byte> name, out ReadOnlySpan<byte> value)
```

Retrieves the cookie at index `i` as zero-copy spans. Returns `false` if the index is out of range.

```csharp
for (var i = 0; i < request.Cookies; i++)
{
    if (request.TryGetCookie(i, out var name, out var value))
    {
        // parsed from Cookie header — e.g. sessionId=abc123
    }
}
```

---

## Building Requests (Client-Side)

`RequestModel` can also be used to construct outgoing HTTP requests. All builder methods are fluent:

### Quick builders

```csharp
// GET
using var req = Lucifer.Rent<RequestModel>();
req.MakeGetRequest("/api/users"u8);

// POST with JSON body
req.MakePostRequest("/api/login"u8, jsonBytes, "application/json"u8);

// PUT
req.MakePutRequest("/api/user/1"u8, jsonBytes);

// DELETE
req.MakeDeleteRequest("/api/user/1"u8);

// HEAD, OPTIONS, TRACE
req.MakeHeadRequest("/api/health"u8);
req.MakeOptionsRequest("/api/user"u8);
req.MakeTraceRequest("/api/user"u8);
```

### Manual builder

For full control, use the fluent `SetBegin → SetHeader → SetCookie → SetBody` chain:

```csharp
using var req = Lucifer.Rent<RequestModel>();
req.SetBegin("POST"u8, "/api/login"u8)
   .SetHeader("Content-Type"u8, "application/json"u8)
   .SetCookie("session"u8, "abc123"u8)
   .AddCookie("lang"u8, "en"u8)
   .SetBody(jsonBytes);
```

### Custom method

```csharp
req.MakeCustomRequest("PATCH"u8, "/api/user/1"u8, jsonBytes, "application/json"u8);
```

---

## Builder Methods Reference

| Method | Description |
|---|---|
| `SetBegin(method, url)` | Sets start-line with default `HTTP/1.1` protocol |
| `SetBegin(method, url, protocol)` | Sets start-line with explicit protocol |
| `SetHeader(key, value)` | Appends a request header |
| `SetCookie(name, value)` | Creates a new `Cookie` header with the first pair |
| `AddCookie(name, value)` | Appends a cookie pair to an existing `Cookie` header. Creates one if none exists |
| `SetBody()` | Appends empty body with `Content-Length: 0` |
| `SetBody(body)` | Appends body bytes and sets `Content-Length` |

All generic parameters (`TMethod`, `TUrl`, `TContent`, etc.) accept both `byte` (UTF-8) and `char` spans.

---

## Clone

```csharp
public RequestModel Clone(bool copyHeaders = true, bool copyCookies = true, bool copyBody = true)
```

Creates a shallow structural copy of the request, rented from the pool. Useful for async processing where the original request may be returned to the pool before processing completes.

```csharp
var clone = request.Clone();
// process clone on a background task
// return when done: Lucifer.Return(clone);
```

---

## Implicit Conversion

```csharp
public static implicit operator ReadOnlySpan<byte>(RequestModel b)
```

Implicitly converts the request to a `ReadOnlySpan<byte>` over the full raw cache buffer. Useful for forwarding or TRACE echo responses.

---

## Lifecycle & Pooling

`RequestModel` instances passed to handler methods are managed by the session layer — do not call `Dispose()` or `Lucifer.Return()` on them unless you cloned them yourself.

```csharp
[HttpPost("")]
protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session)
{
    // Do NOT dispose 'request' here — the framework owns it
    var body = request.BodySpan;
    // ...
}
```

---

## Remarks

- All span properties (`MethodSpan`, `UrlSpan`, `BodySpan`, etc.) are zero-copy views into the internal `Cache` buffer. Never use them after the request is returned to the pool.
- `UrlSpan` includes the full URL with query string. Strip the query string manually or rely on the routing layer which strips it automatically for dispatch key construction.
- Cookie parsing is performed automatically from the `Cookie` header during `ReceiveHeader`. Cookies are stored as position metadata into the same `Cache` buffer.
- Methods `GET`, `HEAD`, `DELETE`, `OPTIONS`, and `TRACE` are treated as body-less — `ReceiveBody` returns immediately with `BodyLength = 0` for these verbs.
