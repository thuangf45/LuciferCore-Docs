# Utf8Builder

**Namespace:** `LuciferCore.Utf8`

`Utf8Builder` is a pooled, fluent UTF-8 builder.

It writes directly to bytes (via pooled `Buffer`) and supports chainable `Append(...)` methods for text, numbers, time, enums, and ANSI colors.

```csharp
public sealed class Utf8Builder : PooledObject<Utf8Builder>
```

Both the builder and its internal buffer are pooled.

---

## Create / rent

```csharp
using var builder = Lucifer.Rent<Utf8Builder>();
builder.Append("Hello, "u8).Append("world"u8);
var s = builder.ToString(); // allocation happens here
```

No constructor overload for initial capacity.  
If needed:

```csharp
builder.Cache.Reserve(capacity);
```

---

## Properties

| Property | Type | Meaning |
|---|---|---|
| `Cache` | `Buffer` | internal pooled buffer (safety-checked) |
| `Size` | `int` | bytes written |
| `Span` | `ReadOnlySpan<byte>` | zero-copy content span |

Also:

```csharp
ReadOnlySpan<byte> Slice(int start, int length)
```

> `Slice(...)` reads from raw `Cache.Data` indexing; in normal usage `Offset` is `0`, so behavior matches expected output.

---

## Append methods

All return `Utf8Builder` for fluent chaining.

### Basic

```csharp
Append(byte b)
Append(char c)
Append(ReadOnlySpan<byte> data)
Append(ReadOnlySpan<char> chars)
Append<T>(ReadOnlySpan<T> data) where T : unmanaged
```

### Numbers / bool / Guid (UTF-8 text output)

```csharp
Append(int)
Append(long)
Append(float)
Append(double)
Append(bool)   // true/false
Append(Guid)
```

### Time

```csharp
AppendTimestamp(DateTime dt) // yyyy-MM-dd HH:mm:ss
AppendTimeSpan(TimeSpan ts)  // hh:mm:ss (keeps sign for negative)
```

### Formatted double

```csharp
AppendFormatted(double value, int decimals = 2)
```

Supports `F0..F4` (outside range falls back to `F2`).

### Enum

```csharp
Append(LogLevel level)
```

Outputs: `INFO`, `WARN`, `ERROR`, `DEBUG`, else `UNKNOWN`.

### ANSI color

```csharp
AppendColor(ConsoleColor color)
ResetColor()
```

Unsupported color values are ignored (no-op).

---

## Utility methods

```csharp
Clear()      // reset internal buffer content
ToString()   // UTF-8 decode to string (allocates)
Dispose()    // return builder + buffer to pool
```

---

## Example

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
```

---

## Notes

- Numeric append writes readable UTF-8 digits, not binary values.
- `Append(char)` UTF-8 encodes directly.
- `ToString()` is the main allocation point.
- After dispose/reset, internal cache is re-rented lazily on next use.
- Enum-specific append currently supports `LogLevel`.