# Utf8Builder

**Namespace:** `LuciferCore.Utf8`

`Utf8Builder` is a pooled, fluent UTF-8 byte string builder. It wraps a pooled `Buffer` and exposes a chainable `Append` API that writes everything directly as UTF-8 bytes — numbers, timestamps, enums, ANSI colors — with zero intermediate allocations.

It is a `sealed class` that extends `PooledObject<Utf8Builder>` (inherits `IDisposable` automatically). Both the builder and its internal `Buffer` are pooled — `Dispose()` returns both to their respective pools. The `Cache` property calls `CheckSafety()` on every access, throwing `ObjectDisposedException` immediately if the builder has already been disposed.

---

## Obtaining an Instance

```csharp
using var builder = Lucifer.Rent<Utf8Builder>();
builder.Append("Hello, "u8).Append("world"u8);
var result = builder.ToString(); // only allocates here
```

There is only a parameterless constructor — `Utf8Builder()` rents a default-capacity `Buffer` internally. There is **no** overload that accepts an initial capacity; if you need a larger starting buffer, call `builder.Cache.Reserve(capacity)` after renting.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Cache` | `Buffer` | The underlying pooled buffer. Getter calls `CheckSafety()` and lazily rents a new `Buffer` if `_cache` is null. Setter is `private` — assigning disposes the previous buffer before storing the new one. |
| `Size` | `int` | Number of bytes currently written (`(int)Cache.Size`). |
| `Span` | `ReadOnlySpan<byte>` | Zero-copy view of the current content (`Cache.AsSpan()`, i.e. the unread region `[Offset, Size)`). |

```csharp
ReadOnlySpan<byte> Slice(int start, int length)
```

> **Note:** `Slice` reads directly from `Cache.Data` (the raw underlying array, indexed from `0`) — not from `Cache.AsSpan()` (which is relative to `Offset`). If `Cache.Offset` is non-zero, `Slice` and `Span`/`ToString()` are addressing different starting points. In normal `Utf8Builder` usage `Offset` stays `0`, so this distinction rarely matters, but it's worth knowing if you manipulate `Cache` directly.

---

## Append Methods

All `Append` overloads return `this` for fluent chaining. All writes are zero-allocation UTF-8.

### Bytes & Chars

```csharp
Utf8Builder Append(byte b)
Utf8Builder Append(char c)                        // UTF-8 encodes the char (via Cache.AppendUtf8)
```

### Numbers (written as UTF-8 decimal text)

```csharp
Utf8Builder Append(int value)
Utf8Builder Append(long value)
Utf8Builder Append(float value)
Utf8Builder Append(double value)
Utf8Builder Append(bool value)    // writes "true" or "false" (delegates to Buffer.AppendUtf8(bool))
Utf8Builder Append(Guid value)    // writes standard GUID format
```

### DateTime & TimeSpan

```csharp
Utf8Builder AppendTimestamp(DateTime dt)   // format: yyyy-MM-dd HH:mm:ss
Utf8Builder AppendTimeSpan(TimeSpan ts)    // format: hh:mm:ss (negative spans get a leading '-')
```

```csharp
builder.AppendTimestamp(DateTime.UtcNow);  // → 2026-03-14 09:30:00
builder.AppendTimeSpan(elapsed);           // → 01:23:45
```

Both use direct digit arithmetic (`Append2Digits`/`Append4Digits` helpers, or zero-padding + `Append(int)` for `TimeSpan`) — no `Format` call, no `stackalloc`.

### Formatted Double

```csharp
Utf8Builder AppendFormatted(double value, int decimals = 2)
```

Writes a double with a fixed number of decimal places (`F0`–`F4`; anything outside `0..4` falls back to `F2`) using `stackalloc char[32]` — no heap allocation in the common path. Falls back to a `ToString()`-based allocation only if `TryFormat` fails (rare).

```csharp
builder.AppendFormatted(3.14159, 3); // → 3.142
```

### Enums

```csharp
Utf8Builder Append(LogLevel level)
// Writes: "INFO", "WARN", "ERROR", "DEBUG", or "UNKNOWN" for any other value
```

### ANSI Colors

```csharp
Utf8Builder AppendColor(ConsoleColor color)  // writes ANSI escape code; unsupported colors are a no-op (returns `this` unchanged)
Utf8Builder ResetColor()                     // writes \u001b[0m
```

```csharp
builder
    .AppendColor(ConsoleColor.Green)
    .Append("OK"u8)
    .ResetColor();
```

Only these 16 `ConsoleColor` values map to escape codes: `Black`, `DarkBlue`, `DarkGreen`, `DarkCyan`, `DarkRed`, `DarkMagenta`, `DarkYellow`, `Gray`, `DarkGray`, `Blue`, `Green`, `Cyan`, `Red`, `Magenta`, `Yellow`, `White`. Any other value (or future additions to `ConsoleColor`) writes nothing.

### Spans

```csharp
Utf8Builder Append(ReadOnlySpan<byte> data)
Utf8Builder Append(ReadOnlySpan<char> chars)       // UTF-8 encodes chars
Utf8Builder Append<T>(ReadOnlySpan<T> data) where T : unmanaged   // generic span, delegates to Cache.Append<T>
```

---

## Utility Methods

```csharp
Utf8Builder Clear()       // calls Cache.Reset() — clears content, may discard/re-rent the backing array if it's oversized
string ToString()         // decode Cache as UTF-8 string (allocates)
void Dispose()            // inherited from PooledObject<Utf8Builder> → Lucifer.Return(this) → triggers Reset()
```

---

## Full Example

```csharp
using var builder = Lucifer.Rent<Utf8Builder>();

builder
    .AppendColor(ConsoleColor.Cyan)
    .Append('[')
    .AppendTimestamp(DateTime.UtcNow)
    .Append("] ["u8)
    .Append(LogLevel.INFO)
    .Append("] "u8)
    .AppendColor(ConsoleColor.White)
    .Append("Server started on port "u8)
    .Append(8443)
    .ResetColor();

Console.WriteLine(builder.ToString());
// [2026-03-14 09:30:00] [INFO] Server started on port 8443
```

---

## Remarks

- All numeric `Append` overloads delegate to `Buffer.AppendUtf8()` — they write human-readable decimal ASCII digits, not binary-encoded integers.
- `Append(char)` encodes to UTF-8 via `Cache.AppendUtf8(char)` — no `string` intermediate.
- `AppendTimestamp` and `AppendTimeSpan` use direct digit arithmetic — safe to call in tight loops.
- `ToString()` is the only method that allocates. Call it once at the end for logging or serialization.
- `Reset()` (called internally by `Dispose()`/pool return) disposes `_cache` and sets it to `null`. The next access to `Cache` after that — whether from a freshly re-rented `Utf8Builder` or otherwise — lazily rents a brand-new `Buffer`, since the constructor only runs once per instance lifetime, not on every rent from the pool.
- There is no `HealthStatus` overload of `Append` — only `LogLevel` is supported among enum types.

---