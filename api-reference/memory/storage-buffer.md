# Buffer

**Namespace:** `LuciferCore.Storage`

`Buffer` is the main pooled byte container in LuciferCore.

It is:
- dynamic size
- append-friendly
- backed by `ArrayPool<byte>`
- used by core models (`PacketModel`, `RequestModel`, `ResponseModel`, `Utf8Builder`, ...)

It inherits `PooledObject<Buffer>` (has `Dispose()`).

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

---

## Notes

- `Append(char span)` encodes UTF-8 directly.
- `AppendUtf8(int)` writes text digits, not binary int bytes.
- `Offset` and `Size` are `long`, but real array still follows CLR limit.
- Keep write/read order clear when combining `Offset`, `Shift`, and `Write`.