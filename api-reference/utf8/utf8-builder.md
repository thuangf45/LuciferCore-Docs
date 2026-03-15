# Utf8Builder

**Namespace:** `LuciferCore.Utf8`

`Utf8Builder` is a pooled, fluent UTF-8 byte string builder. It wraps a pooled `Buffer` and exposes a chainable `Append` API that writes everything directly as UTF-8 bytes — numbers, timestamps, enums, ANSI colors — with zero intermediate allocations.

It extends `PooledObject` and implements `IDisposable`. Both the builder and its internal `Buffer` are pooled — `Dispose()` returns both to their respective pools. The `Cache` property calls `CheckSafety()` on every access, throwing `ObjectDisposedException` immediately if the builder has already been disposed.

---

## Obtaining an Instance

```csharp
using var builder = Lucifer.Rent<Utf8Builder>();
builder.Append("Hello, "u8).Append("world"u8);
var result = builder.ToString(); // only allocates here
```

Or with explicit capacity reservation:

```csharp
using var builder = new Utf8Builder(capacity: 512);
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Size` | `int` | Number of bytes currently written |
| `Span` | `ReadOnlySpan<byte>` | Zero-copy view of the current content |

```csharp
ReadOnlySpan<byte> Slice(int start, int length)
```

---

## Append Methods

All `Append` overloads return `this` for fluent chaining. All writes are zero-allocation UTF-8.

### Bytes & Spans

```csharp
Utf8Builder Append(byte b)
Utf8Builder Append(char c)                        // UTF-8 encodes the char
Utf8Builder Append(ReadOnlySpan<byte> data)
Utf8Builder Append(ReadOnlySpan<char> chars)       // UTF-8 encodes chars
Utf8Builder Append<T>(ReadOnlySpan<T> data)        // generic unmanaged span
```

### Numbers (written as UTF-8 decimal text)

```csharp
Utf8Builder Append(int value)
Utf8Builder Append(long value)
Utf8Builder Append(float value)
Utf8Builder Append(double value)
Utf8Builder Append(bool value)    // writes "True" or "False"
Utf8Builder Append(Guid value)    // writes standard GUID format
```

### DateTime & TimeSpan

```csharp
Utf8Builder AppendTimestamp(DateTime dt)   // format: yyyy-MM-dd HH:mm:ss
Utf8Builder AppendTimeSpan(TimeSpan ts)    // format: hh:mm:ss
```

```csharp
builder.AppendTimestamp(DateTime.UtcNow);  // → 2026-03-14 09:30:00
builder.AppendTimeSpan(elapsed);           // → 01:23:45
```

### Formatted Double

```csharp
Utf8Builder AppendFormatted(double value, int decimals = 2)
```

Writes a double with a fixed number of decimal places using `stackalloc` — no heap allocation.

```csharp
builder.AppendFormatted(3.14159, 3); // → 3.142
```

### Enums

```csharp
Utf8Builder Append(LogLevel level)
// Writes: "INFO", "WARN", "ERROR", "DEBUG"

Utf8Builder Append(HealthStatus status)
// Writes: "Healthy", "Degraded", "Unhealthy", "Disposed", etc.
```

### ANSI Colors

```csharp
Utf8Builder AppendColor(ConsoleColor color)  // writes ANSI escape code
Utf8Builder ResetColor()                     // writes \u001b[0m
```

```csharp
builder
    .AppendColor(ConsoleColor.Green)
    .Append("OK"u8)
    .ResetColor();
```

All 16 `ConsoleColor` values are supported with their standard ANSI escape codes.

---

## Utility Methods

```csharp
Utf8Builder Clear()       // reset buffer, keep capacity
string ToString()         // decode as UTF-8 string (allocates)
void Dispose()            // Lucifer.Return(this) → triggers Reset() → Buffer returned to pool
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
- `Append(char)` encodes to UTF-8 in-place — no `string` intermediate.
- `AppendTimestamp` and `AppendTimeSpan` use direct digit arithmetic (no `Format` call, no `stackalloc` overhead) — they are safe to call in tight loops.
- `ToString()` is the only method that allocates. Call it once at the end for logging or serialization.
- `Dispose()` calls `Lucifer.Return(this)`. The pool's `Return` decrements `RefCount` to `0`, which triggers `Reset()` — `Reset()` is where the internal `Buffer` is returned to its own pool (`_cache` is returned and nulled). This means the `Buffer` is released as part of the normal pool return cycle, not as a separate explicit call in `Dispose()`.
