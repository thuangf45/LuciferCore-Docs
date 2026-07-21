# Buffer

**Namespace:** `LuciferCore.Memory`

`Buffer` is the main pooled byte container in LuciferCore.

It is:
- dynamic size
- append-friendly
- backed by `ArrayPool<byte>`
- used by core models (`PacketModel`, `RequestModel`, `ResponseModel`, `Utf8Builder`, ...)

It inherits `PooledObject<Buffer>` (has `Dispose()`).

`Buffer` is declared `partial`; static span/binary utilities live in a separate partial file (`Buffer.Binary.cs`) — see [Static binary utilities](#static-binary-utilities-bufferbinarycs) below.

Use:

```csharp
using var buffer = Lucifer.Rent<Buffer>();
```

---

## Capacity limits

| Member | Value | Meaning |
|---|---|---|
| `MaxBufferCapacity` | `0x7FFFFFC7` | hard max capacity |
| `DefaultCapacity` | `256` (default) | default new capacity |
| `MaxRetainedCapacity` | `262144` (default) | if bigger, reset will drop big array |
| `MaxSupportedCapacity` | same as max | exposed property |

If buffer is too large on reset, internal array is replaced with smaller default one.

---

## Main properties

| Property | Type | Meaning |
|---|---|---|
| `Data` | `byte[]` | underlying array |
| `Size` | `long` | written bytes count |
| `Offset` | `long` | read cursor |
| `Capacity` | `long` | array length |
| `IsEmpty` | `bool` | empty state |
| `IsValid` | `bool` | offset/size validity |

---

## Indexers

```csharp
byte this[long index]
byte[] this[Range range]
```

---

## Span / memory / stream views

```csharp
AsSpan()
AsSpan(start, length)
AsMemory()
AsMemory(start, length)
AsSegment()
Slice(start, length)
AsStream(writable)
AsStream(start, length, writable)
AsWritableStream()
```

Implicit conversion:

```csharp
ReadOnlySpan<byte> span = buffer;
```

---

## Struct helpers

```csharp
ref T InitStruct<T>() where T : unmanaged
ref T AsStruct<T>() where T : unmanaged
ref T AsStruct<T>(long offset) where T : unmanaged
```

Useful for binary protocol layouts.

---

## Append methods

```csharp
Append<T>(T value)
Append<T>(ReadOnlySpan<T> data)
```

Notes:
- `byte` span -> copied directly
- `char` span -> encoded UTF-8
- other unmanaged types -> raw bytes

---

## `AppendUtf8(...)` methods

Write values as UTF-8 text (human-readable), e.g. for HTTP headers.

Examples:
- `AppendUtf8(int)`
- `AppendUtf8(long)`
- `AppendUtf8(bool)` (`true` / `false`)
- `AppendUtf8(Guid)`
- `AppendUtf8(float/double/decimal)`

---

## Write methods (at offset)

```csharp
Write<T>(long offset, T value, bool force = true)
Write<T>(long offset, ReadOnlySpan<T> data, bool force = true)
```

- `force = true`: overwrite
- `force = false`: insert (shift data to open gap)

---

## Read methods

```csharp
Read<T>(long offset)
Read<T>(long offset, long count)
```

For UTF-8 text, prefer `ExtractString(...)`.

---

## Memory management methods

```csharp
Reserve(capacity)
Resize(size)
Remove(offset, size)
Clear()
Reset()
Attach()
Attach(capacity)
Attach(byte[] buffer, long offset, long size)
```

Important:
- `Clear()` keeps array
- `Reset()` may replace too-large array
- `Attach()` no-arg leaves empty unpooled array
- `Attach(capacity)` rents fresh pooled array

---

## Shift / Unshift

### 1-arg (move read cursor)

```csharp
Shift(offset)
Unshift(offset)
```

### 2-arg (move data content)

```csharp
Shift(offset, size)
Unshift(offset, size)
```

Do not confuse cursor move vs content move.

---

## Utility methods

```csharp
ExtractString(offset, size)
ToString()
GetBytesCount<T>(span)
SafeToInt32(long value)
```

---

## Static binary utilities (`Buffer.Binary.cs`)

These are `static` helper methods on `Buffer`, operating directly on `ReadOnlySpan<T>`/`Span<T>` — they do **not** touch an instance's `Data`/`Size`/`Offset`. Useful for parsing protocols (HTTP headers, delimited frames, etc.) without allocating.

### Copy

```csharp
static void Copy(ReadOnlySpan<byte> source, Span<byte> destination)
static bool TryCopy(ReadOnlySpan<byte> source, Span<byte> destination)
```

- `Copy` throws if destination is too small; `TryCopy` returns `false` instead.
- Safe for overlapping spans (same underlying array).

### IndexOf / LastIndexOf

```csharp
static int IndexOf<T>(ReadOnlySpan<T> span, T value, bool ignoreCase = false)
    where T : unmanaged, IEquatable<T>
static int IndexOf(ReadOnlySpan<byte> span, ReadOnlySpan<byte> value, bool ignoreCase = false)

static int LastIndexOf<T>(ReadOnlySpan<T> span, T value, bool ignoreCase = false)
    where T : unmanaged, IEquatable<T>
static int LastIndexOf(ReadOnlySpan<byte> span, ReadOnlySpan<byte> value, bool ignoreCase = false)
```

- `ignoreCase` is only meaningful for `byte` (ASCII) and `char` (via `StringComparison.OrdinalIgnoreCase`); other unmanaged `T` fall back to exact binary comparison.
- Byte-level ignore-case search is SIMD-accelerated (`IndexOfAny`/`LastIndexOfAny` over both letter cases) — no per-character allocation.

### Contains

```csharp
static bool Contains<T>(ReadOnlySpan<T> span, T value, bool ignoreCase = false)
    where T : unmanaged, IEquatable<T>
static bool Contains(ReadOnlySpan<byte> span, ReadOnlySpan<byte> value, bool ignoreCase = false)
```

Thin wrapper over `IndexOf(...) >= 0`.

### StartsWith / EndsWith

```csharp
static bool StartsWith<T>(ReadOnlySpan<T> span, ReadOnlySpan<T> value, bool ignoreCase = false)
    where T : unmanaged
static bool EndsWith<T>(ReadOnlySpan<T> span, ReadOnlySpan<T> value, bool ignoreCase = false)
    where T : unmanaged
```

- `byte`/`char` get proper case-insensitive comparison; other `T` fall back to exact binary comparison even if `ignoreCase = true`.

### TrySplitAt

```csharp
static bool TrySplitAt<T>(ReadOnlySpan<T> span, T delimiter,
    out ReadOnlySpan<T> before, out ReadOnlySpan<T> after, bool ignoreCase = false)
    where T : unmanaged, IEquatable<T>

static bool TrySplitAt(ReadOnlySpan<byte> span, ReadOnlySpan<byte> delimiter,
    out ReadOnlySpan<byte> before, out ReadOnlySpan<byte> after, bool ignoreCase = false)
```

- Splits at the **first** occurrence of the delimiter.
- If not found: `before` = the whole span, `after` = empty, returns `false`.
- `after` excludes the delimiter itself.

Example:

```csharp
Buffer.TrySplitAt(line, (byte)':', out var key, out var value);
```

### Equals / Compare

```csharp
static bool Equals<T>(ReadOnlySpan<T> left, ReadOnlySpan<T> right, bool ignoreCase = false)
    where T : unmanaged, IEquatable<T>
static bool Equals(byte b, ReadOnlySpan<byte> span, bool ignoreCase = false)
static bool Equals(ReadOnlySpan<byte> span, byte b, bool ignoreCase = false)
static bool Equals(ReadOnlySpan<byte> bytes, ReadOnlySpan<char> chars, bool ignoreCase = false)
static bool Equals(ReadOnlySpan<char> chars, ReadOnlySpan<byte> bytes, bool ignoreCase = false)

static int Compare<T>(ReadOnlySpan<T> left, ReadOnlySpan<T> right, bool ignoreCase = false)
    where T : unmanaged, IComparable<T>
```

- The `byte` vs `char` overloads let you compare a raw UTF-8 buffer directly against a string/char span without manually decoding — internally encodes `chars` to UTF-8 (stack for ≤256 bytes, pooled array otherwise) and compares bytes.
- `Compare` returns negative/zero/positive like `IComparable`; `ignoreCase` only affects `byte`/`char`.

### ComputeHash

```csharp
static int ComputeHash<T>(ReadOnlySpan<T> span, bool ignoreCase = false)
    where T : unmanaged
```

- Uses an XXH64-based algorithm, folded down to a 32-bit `int`.
- `char` spans are UTF-8 encoded first (stack for ≤512 bytes, pooled array otherwise), then hashed.
- `ignoreCase = true` normalizes ASCII letters (`A-Z` → `a-z`) before hashing; non-ASCII bytes pass through unchanged. Useful for case-insensitive dictionary keys (e.g. HTTP header names) without allocating a `string`.
- Not a cryptographic hash — do not use for security purposes (signatures, password hashing, etc.).

### Other

```csharp
static ReadOnlySpan<byte> Trim(ReadOnlySpan<byte> span)
static bool IsWhitespace(byte b)
static byte ToLower(byte b)
```

- `Trim` strips leading/trailing space, tab, `\n`, `\r` (ASCII whitespace only).
- `ToLower` is a branchless ASCII-only lowercase conversion (`A-Z` → `a-z`); non-letter bytes pass through unchanged.

---

## Example

```csharp
using var buf = Lucifer.Rent<Buffer>();
buf.Append("HTTP/1.1 200 OK\r\n"u8);
buf.Append("Content-Length: "u8);
buf.AppendUtf8(body.Length);
buf.Append("\r\n\r\n"u8);
buf.Append(body);

session.SendResponseAsync(buf.AsSpan());
```

Parsing example using the new static utilities:

```csharp
ReadOnlySpan<byte> line = "Content-Type: application/json"u8;

if (Buffer.TrySplitAt(line, (byte)':', out var key, out var value))
{
    var trimmedKey = Buffer.Trim(key);
    var trimmedValue = Buffer.Trim(value);

    if (Buffer.Equals(trimmedKey, "content-type"u8, ignoreCase: true))
    {
        // matched header, case-insensitive, zero string allocation
    }
}
```

---

## Notes

- `Append(char span)` encodes UTF-8 directly.
- `AppendUtf8(int)` writes text digits, not binary int bytes.
- `Offset` and `Size` are `long`, but real array still follows CLR limit.
- Keep write/read order clear when combining `Offset`, `Shift`, and `Write`.
- The static utilities in `Buffer.Binary.cs` are pure span operations — they can be called without renting/holding a `Buffer` instance at all (e.g. `Buffer.Equals(...)`, `Buffer.ComputeHash(...)`).
- `ignoreCase` support is limited to `byte` and `char` element types; other unmanaged `T` always compare/search as exact binary regardless of the flag.