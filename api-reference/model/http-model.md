# HttpModel

**Namespace:** `LuciferCore.Model`

`HttpModel` is the abstract base class for all HTTP data structures in LuciferCore. It owns a single pooled `Buffer` (the *cache*) and describes any HTTP message as three invariant parts:

```
┌─────────────────────────────────────────────────────┐
│  Start-line:  FirstSpan  SecondSpan  ThirdSpan      │
│  Headers:     key₀: value₀  …  keyₙ: valueₙ          │
│  Body:        BodySpan                              │
└─────────────────────────────────────────────────────┘
```

The three start-line tokens are **position-based, not semantically named** at this level. What each token *means* is defined entirely by the subclass:

| Subclass | FirstSpan | SecondSpan | ThirdSpan |
|---|---|---|---|
| `RequestModel` | Method (`GET`, `POST`, …) | URL (`/api/user`) | Protocol (`HTTP/1.1`) |
| `ResponseModel` | Protocol (`HTTP/1.1`) | Status code (`200`) | Status phrase (`OK`) |
| Custom model | anything | anything | anything |

All content lives inside a single `Buffer`. Every property that exposes content returns a `ReadOnlySpan<byte>` — a **zero-copy view** into that buffer. There are no intermediate strings or extra heap allocations at read time.

`HttpModel` extends `PooledObject` and implements `IDisposable`. Subclasses inherit automatic pool lifecycle management.

```csharp
public class HttpModel : PooledObject, IDisposable
```

---

## Properties

### Start-line Spans

| Property | Type | Description |
|---|---|---|
| `FirstSpan` | `ReadOnlySpan<byte>` | Zero-copy span of the first start-line token |
| `SecondSpan` | `ReadOnlySpan<byte>` | Zero-copy span of the second start-line token |
| `ThirdSpan` | `ReadOnlySpan<byte>` | Zero-copy span of the third start-line token |

All three are bounds-checked. If the position stored for a span extends past the current cache size, an empty span is returned instead of throwing.

### Body

| Property | Type | Description |
|---|---|---|
| `BodySpan` | `ReadOnlySpan<byte>` | Zero-copy span of the body content |
| `BodyLength` | `int` | Expected body length in bytes as declared by `Content-Length` |

### Metadata

| Property | Type | Description |
|---|---|---|
| `Headers` | `int` | Number of header entries currently stored |
| `Cookies` | `int` | Number of cookie entries currently stored |
| `IsEmpty` | `bool` | `true` if the cache buffer contains no data |
| `IsErrorSet` | `bool` | `true` if a parse error was recorded during `ReceiveHeader` |
| `Cache` | `Buffer` | The underlying pooled buffer. Lazy-initialized on first access. Calling `CheckSafety()` on a disposed instance throws `ObjectDisposedException` |

---

## Methods

### `SetBegin<TFirst, TSecond, TThird>(first, second, third)`

```csharp
public virtual HttpModel SetBegin<TFirst, TSecond, TThird>(
    ReadOnlySpan<TFirst> first,
    ReadOnlySpan<TSecond> second,
    ReadOnlySpan<TThird> third)
    where TFirst : unmanaged
    where TSecond : unmanaged
    where TThird : unmanaged
```

Writes the three start-line tokens into the cache, separated by single space bytes, and terminated with `CRLF`. Calls `Clear()` first — calling `SetBegin` on an already-populated model resets all existing content.

Accepts `byte` (UTF-8 literals) or `char` spans for every token. Mixed element types across parameters are allowed.

### `SetHeader<TKey, TValue>(key, value)`

```csharp
public virtual HttpModel SetHeader<TKey, TValue>(
    ReadOnlySpan<TKey> key,
    ReadOnlySpan<TValue> value)
    where TKey : unmanaged
    where TValue : unmanaged
```

Appends a `key: value\r\n` header line to the cache and records its position metadata. Must be called before `SetBody`.

### `AddHeader<TKey, TValue>(key, value)`

```csharp
public virtual HttpModel AddHeader<TKey, TValue>(
    ReadOnlySpan<TKey> key,
    ReadOnlySpan<TValue> value)
    where TKey : unmanaged
    where TValue : unmanaged
```

Inserts a header **before the body** if a body has already been written, by shifting cache bytes forward at the insert position. Falls back to `SetHeader` if no body exists yet.

### `SetContentType<TExt>(extension)`

```csharp
public virtual HttpModel SetContentType<TExt>(ReadOnlySpan<TExt> extension)
    where TExt : unmanaged
```

Looks up `extension` in LuciferCore's built-in MIME table and calls `SetHeader` with `Content-Type`. No-ops if the extension is not found.

```csharp
model.SetContentType(".json"u8); // Content-Type: application/json
model.SetContentType(".html"u8); // Content-Type: text/html
```

### `SetBody()` / `SetBody<T>(body)`

```csharp
public virtual HttpModel SetBody()
public virtual HttpModel SetBody<T>(ReadOnlySpan<T> body) where T : unmanaged
```

Appends `\r\n` (end-of-headers), then the body content. Automatically calculates byte length (handles `char` → UTF-8 byte counting) and writes a `Content-Length` header. Call this **after** all `SetHeader` calls.

### `TryGetHeader(i, out key, out value)`

```csharp
public bool TryGetHeader(int i, out ReadOnlySpan<byte> key, out ReadOnlySpan<byte> value)
```

Returns the header at zero-based index `i` as zero-copy spans. Returns `false` and empty spans if the index is out of range.

### `TryGetCookie(i, out name, out value)`

```csharp
public bool TryGetCookie(int i, out ReadOnlySpan<byte> name, out ReadOnlySpan<byte> value)
```

Returns the cookie at zero-based index `i` as zero-copy spans. Returns `false` and empty spans if the index is out of range.

### `Clone<T>(...)`

```csharp
public T Clone<T>(
    bool copyHeaders = true,
    bool copyCookies = true,
    bool copyBody = true)
    where T : HttpModel, new()
```

Rents a new `T` from the pool and deep-copies the cache buffer and position metadata into it. Partial copies are supported — set any flag to `false` to skip that section.

### `Clear()`

```csharp
public virtual HttpModel Clear()
```

Resets all position metadata and clears the cache buffer. Does **not** return the cache buffer to the pool — the `HttpModel` instance retains its buffer for reuse.

### `Dispose()`

Calls `Lucifer.Return(this)`, returning the instance and its internal `Buffer` to their respective object pools.

---

## Implicit Conversion

```csharp
public static implicit operator ReadOnlySpan<byte>(HttpModel b)
```

Returns a `ReadOnlySpan<byte>` over the entire raw cache. Used internally by session send methods to pass the serialized message directly.

---

## Overridable Hooks

Subclasses can override these to extend parsing behavior:

| Method | Description |
|---|---|
| `ParseCookieHeader(value, full)` | Called during `ReceiveHeader` when a `Cookie` or `Set-Cookie` header line is encountered. Override to parse cookie pairs out of the header value |
| `OnHeaderParsed()` | Called once after all headers have been parsed. Override to extract derived fields — for example, `ResponseModel` reads `SecondSpan` here to populate its `Status` integer |

---

## Wire Format

`HttpModel` writes and parses the standard HTTP/1.1 wire format:

```
{First} SP {Second} SP {Third} CRLF
{HeaderKey}: {HeaderValue} CRLF
…
CRLF
{Body}
```

The only structural variation between a request and a response is **which token occupies which position** in the start-line — the rest of the layout is identical.

---

## Remarks

- All span properties are zero-copy views into `Cache`. Never use them after the model is disposed or returned to the pool.
- `SetBegin` calls `Clear()` unconditionally. Do not call `SetBegin` on a model that already has headers or a body you intend to keep.
- `Content-Length` is computed and written automatically by `SetBody`. Do not set it manually.
- `AddHeader` after `SetBody` performs a mid-buffer byte-shift. It is correct but has linear cost proportional to body size. Prefer calling all `SetHeader` before `SetBody`.
- Cookie parsing is handled automatically during `ReceiveHeader`. Override `ParseCookieHeader` in a subclass to customize extraction behavior.
