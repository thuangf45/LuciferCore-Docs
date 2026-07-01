# HttpModel

**Namespace:** `LuciferCore.Model`

`HttpModel` is the base type for HTTP models in LuciferCore.

It stores all HTTP bytes in one pooled `Buffer` and reads data using spans (zero-copy).

```csharp
public class HttpModel : PooledObject
```

Builder/fluent methods are on generic subtype:

```csharp
public partial class HttpModel<T> : HttpModel, IDisposable where T : HttpModel<T>
```

---

## Core idea

An HTTP message has 3 parts:

1. Start-line (3 tokens)
2. Headers
3. Body

At `HttpModel` level, start-line is generic:

- `FirstSpan`
- `SecondSpan`
- `ThirdSpan`

Meaning depends on subtype:

| Type | First | Second | Third |
|---|---|---|---|
| `RequestModel` | Method | URL | Protocol |
| `ResponseModel` | Protocol | Status code | Status text |

---

## Main properties (`HttpModel`)

### Start-line spans

| Property | Type | Meaning |
|---|---|---|
| `FirstSpan` | `ReadOnlySpan<byte>` | First token |
| `SecondSpan` | `ReadOnlySpan<byte>` | Second token |
| `ThirdSpan` | `ReadOnlySpan<byte>` | Third token |

### Position metadata

| Property | Type | Meaning |
|---|---|---|
| `First` | `Position` | First token position |
| `Second` | `Position` | Second token position |
| `Third` | `Position` | Third token position |
| `Body` | `Position` | Body position |

### Body and remain

| Property | Type | Meaning |
|---|---|---|
| `BodySpan` | `ReadOnlySpan<byte>` | Body bytes |
| `Remain` | `ReadOnlySpan<byte>` | Bytes after body |
| `BodyLength` | `int` | Body length |
| `RemainLength` | `int` | Remaining bytes count |

### Metadata

| Property | Type | Meaning |
|---|---|---|
| `Headers` | `int` | Header count |
| `Cookies` | `int` | Cookie count |
| `IsEmpty` | `bool` | Buffer empty or not |
| `IsErrorSet` | `bool` | Parse error flag |
| `AllocHeaderOption` | `bool` | Header lookup cache option |
| `Cache` | `Buffer` | Internal pooled buffer |

---

## Main methods (`HttpModel`)

| Method | Meaning |
|---|---|
| `TryGetHeader(i, out key, out value)` | Get header by index |
| `TryGetHeader<T>(name, out value)` | Get header by name |
| `TryGetCookie(i, out name, out value)` | Get cookie by index |
| `TryGetCookie<T>(name, out value)` | Get cookie by name |
| `Clear()` | Reset model state/buffer content |
| `ParseFull(data)` | Parse one full HTTP message |
| `SplitToModels(body, list)` | Split concatenated HTTP messages |
| `AbsoluteOffset(full, slice)` | Get slice offset in full span |

---

## Overridable hooks (`HttpModel`)

| Method | Meaning |
|---|---|
| `ParseCookieHeader(value, full)` | Custom cookie parse logic |
| `OnHeaderParsed()` | Post-header parse hook |

---

## Fluent builder methods (`HttpModel<T>`)

These are on generic subtype (`RequestModel`, `ResponseModel`, ...), not on plain `HttpModel`.

| Method | Meaning |
|---|---|
| `SetBegin(...)` | Set start-line |
| `SetHeader(...)` | Append header |
| `AddHeader(...)` | Insert header (before body if needed) |
| `SetContentType(...)` | Set `Content-Type` from extension |
| `SetBody(...)` | Set body and auto `Content-Length` |
| `Clone(...)` | Clone into pooled instance |
| `Dispose()` | Return instance to pool |

---

## `SetBody` note

`SetBody(...)` automatically:
- writes `\r\n` header terminator
- writes body
- sets `Content-Length`

So you usually should not set `Content-Length` manually.

---

## Implicit conversion

```csharp
public static implicit operator ReadOnlySpan<byte>(HttpModel b)
```

You can use model directly as raw byte span.

---

## Wire format

```text
{First} SP {Second} SP {Third} CRLF
{HeaderKey}: {HeaderValue} CRLF
...
CRLF
{Body}
```

---

## Notes

- Span properties are zero-copy views into internal buffer.
- Do not use spans after object is returned to pool.
- `SetBegin()` clears previous data first.
- Prefer setting headers before `SetBody()` for best performance.
- `HttpModel<T>` handles pooling/dispose flow.