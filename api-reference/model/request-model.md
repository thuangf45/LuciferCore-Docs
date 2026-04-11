# RequestModel

**Namespace:** `LuciferCore.Model`

`RequestModel` is the concrete `HttpModel` subclass for HTTP requests. It maps the three start-line tokens to the standard request layout and adds request-specific builder and parsing methods.

```
Start-line:  METHOD  SP  URL  SP  HTTP/1.1  CRLF
```

| Token | `HttpModel` span | Exposed as |
|---|---|---|
| Method | `FirstSpan` | `MethodSpan` |
| URL | `SecondSpan` | `UrlSpan` |
| Protocol | `ThirdSpan` | `ProtocolSpan` |

It also implements `IRoutable`, exposing `MethodSpan` and `UrlSpan` to the dispatch pipeline directly.

`RequestModel` extends `PooledObject` and implements `IDisposable`. Instances passed to handler methods are managed by the session layer — do not dispose them unless you cloned them yourself.

```csharp
public class RequestModel : HttpModel, IRoutable
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `MethodSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the HTTP method (e.g. `GET`, `POST`) |
| `UrlSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the full request URL including query string |
| `ProtocolSpan` | `ReadOnlySpan<byte>` | Zero-copy UTF-8 span of the protocol version (e.g. `HTTP/1.1`) |
| `BodySpan` | `ReadOnlySpan<byte>` | Zero-copy span of the request body — inherited from `HttpModel` |
| `Headers` | `int` | Number of parsed headers — inherited from `HttpModel` |
| `Cookies` | `int` | Number of parsed cookies — inherited from `HttpModel` |
| `BodyLength` | `int` | Expected body length as declared by `Content-Length` — inherited |
| `IsEmpty` | `bool` | `true` if the internal cache buffer is empty — inherited |
| `IsErrorSet` | `bool` | `true` if a parse error occurred during header parsing — inherited |

> `MethodRoute` and `UrlRoute` are the `IRoutable` projections of `MethodSpan` and `UrlSpan` respectively. They are read by the dispatch pipeline — do not use them directly in handler code.

---

## Reading Incoming Requests

### Headers

```csharp
for (var i = 0; i < request.Headers; i++)
{
    if (request.TryGetHeader(i, out var key, out var value))
    {
        // key and value are zero-copy ReadOnlySpan<byte> views
    }
}
```

### Cookies

```csharp
for (var i = 0; i < request.Cookies; i++)
{
    if (request.TryGetCookie(i, out var name, out var value))
    {
        // parsed automatically from the Cookie header
    }
}
```

### Body

```csharp
var body = request.BodySpan; // ReadOnlySpan<byte>
```

---

## Building Outgoing Requests (Client-Side)

`RequestModel` doubles as a request builder for client-side use. All methods are fluent and return `this`.

### Quick builders

```csharp
using var req = Lucifer.Rent<RequestModel>();

req.MakeGetRequest("/api/users"u8);
req.MakeHeadRequest("/api/health"u8);
req.MakeDeleteRequest("/api/user/1"u8);
req.MakeOptionsRequest("/api/user"u8);
req.MakeTraceRequest("/api/user"u8);

req.MakePostRequest("/api/login"u8, jsonBytes);
req.MakePostRequest("/api/login"u8, jsonBytes, "application/json"u8);

req.MakePutRequest("/api/user/1"u8, jsonBytes);
req.MakePutRequest("/api/user/1"u8, jsonBytes, "application/json"u8);

req.MakeCustomRequest("PATCH"u8, "/api/user/1"u8, jsonBytes, "application/json"u8);
```

### Manual builder

For full control, chain the inherited `SetBegin → SetHeader → SetCookie → SetBody` methods:

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

## Builder Methods

| Method | Description |
|---|---|
| `SetBegin(method, url)` | Sets start-line with default `HTTP/1.1` protocol |
| `SetBegin(method, url, protocol)` | Sets start-line with explicit protocol |
| `SetHeader(key, value)` | Appends a request header — inherited |
| `AddHeader(key, value)` | Inserts a header before the body — inherited |
| `SetContentType(extension)` | Sets `Content-Type` from MIME table — inherited |
| `SetCookie(name, value)` | Creates a new `Cookie` header with the first pair |
| `AddCookie(name, value)` | Appends a pair to an existing `Cookie` header; creates one if none exists |
| `SetBody()` | Appends an empty body with `Content-Length: 0` — inherited |
| `SetBody(body)` | Appends body content and sets `Content-Length` — inherited |

All generic parameters accept both `byte` (UTF-8) and `char` spans.

---

## Cookie Building Detail

`SetCookie` and `AddCookie` manage a single `Cookie` header line:

```csharp
req.SetCookie("session"u8, "abc123"u8);   // Cookie: session=abc123\r\n
req.AddCookie("lang"u8, "en"u8);          // Cookie: session=abc123; lang=en\r\n
```

`AddCookie` checks `_hasCookieHeader` internally. If no `Cookie` header exists yet, it delegates to `SetCookie`.

---

## Clone

```csharp
public RequestModel Clone(bool copyHeaders = true, bool copyCookies = true, bool copyBody = true)
```

Rents a new `RequestModel` from the pool and deep-copies the cache buffer and position metadata. Useful for deferring processing to a background task where the original request may be returned to the pool before the task completes.

```csharp
var clone = request.Clone();
_ = Task.Run(() =>
{
    // process clone
    Lucifer.Return(clone);
});
```

---

## Implicit Conversion

```csharp
public static implicit operator ReadOnlySpan<byte>(RequestModel b)
```

Converts the model to a `ReadOnlySpan<byte>` over the entire raw cache. Useful for forwarding and TRACE echo:

```csharp
response.MakeTraceResponse(request); // passes request.Cache implicitly
```

---

## Lifecycle & Pooling

Instances arriving in handler methods are owned by the session layer:

```csharp
[HttpPost("")]
protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session)
{
    // Do NOT dispose 'request' — the framework owns this instance
    var body = request.BodySpan;
}
```

Only dispose instances you explicitly cloned or rented:

```csharp
using var req = Lucifer.Rent<RequestModel>();
req.MakeGetRequest("/api/users"u8);
client.SendRequest(req);
```

---

## Remarks

- `UrlSpan` includes the full URL with query string. The routing layer strips the query string automatically during dispatch key construction. Strip it manually if you need the clean path inside a handler.
- `MethodSpan`, `UrlSpan`, `BodySpan`, and all other span properties are zero-copy views into the internal `Cache` buffer. Do not store or use them after the request is disposed or returned to the pool.
- Methods `GET`, `HEAD`, `DELETE`, `OPTIONS`, and `TRACE` are treated as body-less by the receive pipeline — `BodyLength` is set to `0` for these verbs regardless of any incoming bytes.
- Cookie entries are parsed automatically from the `Cookie` header during `ReceiveHeader`. Each `name=value` pair becomes a separate entry in the cookie list.
