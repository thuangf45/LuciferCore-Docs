# Buffer

**Namespace:** `LuciferCore.Storage`

`Buffer` is LuciferCore's core pooled byte container. It is a dynamically expandable, append-optimized byte buffer backed by a rented array from `ArrayPool<byte>`. All hot-path types — `PacketModel`, `RequestModel`, `ResponseModel`, `Utf8Builder` — store their data in a `Buffer`.

It extends `PooledObject<Buffer>` (inherits `IDisposable` automatically). Obtain instances via `Lucifer.Rent<Buffer>()`.

**Capacity limits:** max `MaxBufferCapacity` (`0x7FFFFFC7` = 2,147,483,591 bytes, just under the CLR's 2 GB array limit). Buffers larger than `MaxRetainedCapacity` (default 256 KB) are not retained in the pool on `Reset()` — their backing array is dropped and a fresh default-capacity array is rented instead.

---

## Constants & Static Fields

| Member | Type | Value | Description |
|---|---|---|---|
| `MaxBufferCapacity` | `const long` | `0x7FFFFFC7` (2,147,483,591) | Hard ceiling on buffer capacity. `Reserve()` throws if exceeded. |
| `DefaultCapacity` | `static int` | `256` | Capacity used when attaching a fresh buffer with no explicit size. Mutable. |
| `MaxRetainedCapacity` | `static int` | `262144` (256 KB) | Threshold above which `Reset()` discards the array instead of reusing it. Mutable. |
| `MaxSupportedCapacity` | `static long` (property) | `MaxBufferCapacity` | Same value as `MaxBufferCapacity`, exposed as a property. |

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Data` | `byte[]` | The underlying pooled byte array. Getter and setter both call `CheckSafety()` — accessing `Data` on a disposed `Buffer` throws `ObjectDisposedException`. Setter is private. |
| `Size` | `long` | Logical size — number of bytes written. |
| `Offset` | `long` | Current read position within `Data`. |
| `Capacity` | `long` | Physical size of the underlying array (`Data.Length`). |
| `IsEmpty` | `bool` | `true` if `Data` is null or `Size == 0`. |
| `IsValid` | `bool` | `true` if `Data` is non-null and `0 <= Offset <= Size <= Data.Length`. |

---

## Indexers

```csharp
byte this[long index]        // byte at index, relative to Offset (within [0, Size - Offset))
byte[] this[Range range]     // slice copy of the full underlying Data array via Range syntax
```

---

## Span / Memory / Stream Views

Views over the unread region `[Offset, Size)` unless a `start`/`length` is given:

```csharp
Span<byte>          AsSpan()
Span<byte>          AsSpan(long start, long length)
Memory<byte>        AsMemory()
Memory<byte>        AsMemory(long start, long length)
ArraySegment<byte>  AsSegment()
ReadOnlySpan<byte>  Slice(long start, long length)
Stream               AsStream(bool writable = false)
Stream               AsStream(long start, long length, bool writable = false)
Stream               AsWritableStream()   // writable stream over [Size, Capacity)
```

Implicit conversion to `ReadOnlySpan<byte>`:

```csharp
ReadOnlySpan<byte> span = buffer; // implicit, equivalent to buffer.AsSpan()
```

### Struct views

```csharp
ref T InitStruct<T>() where T : unmanaged          // resets Offset to 0, resizes buffer to sizeof(T), returns ref to it
ref T AsStruct<T>() where T : unmanaged             // reinterprets [Offset, Offset+sizeof(T)) as ref T
ref T AsStruct<T>(long offset) where T : unmanaged  // reinterprets at a given offset as ref T
```

---

## Append Methods

All append/write methods grow the underlying array automatically via `Reserve()` as needed.

```csharp
long Append<T>(T value) where T : unmanaged          // appends raw bytes of any unmanaged value (e.g. int, long, float, bool, Guid...)
long Append<T>(ReadOnlySpan<T> data) where T : unmanaged   // generic span; byte/char get special-cased, others treated as raw bytes
```

`char` spans are UTF-8 encoded in place (no intermediate `string` allocation); `byte` spans are copied directly.

### UTF-8 text overloads (human-readable digits)

For writing numbers as ASCII text into HTTP responses/requests:

```csharp
long AppendUtf8(int value)     long AppendUtf8(long value)
long AppendUtf8(uint value)    long AppendUtf8(ulong value)
long AppendUtf8(float value)   long AppendUtf8(double value)
long AppendUtf8(decimal value) long AppendUtf8(bool value)     // writes "true" / "false"
long AppendUtf8(short value)   long AppendUtf8(ushort value)
long AppendUtf8(sbyte value)   long AppendUtf8(byte value)
long AppendUtf8(Guid value)    long AppendUtf8(char value)     // ASCII fast path, falls back to UTF-8 encode for non-ASCII
```

---

## Write At Methods

Write at an explicit offset rather than appending at `Size`:

```csharp
long Write<T>(long offset, T value, bool force = true) where T : unmanaged
long Write<T>(long offset, ReadOnlySpan<T> data, bool force = true) where T : unmanaged
```

- `force = true` (default): overwrites bytes at `offset`, growing `Size` if the write extends past it.
- `force = false`: first calls `Shift(offset, length)` to open a gap at `offset` (shifting existing bytes right), then writes into the gap — i.e. an insert rather than an overwrite.

---

## Read At Methods

```csharp
T Read<T>(long offset) where T : unmanaged
ReadOnlySpan<T> Read<T>(long offset, long count) where T : unmanaged
```

`Read<T>(offset, count)` throws `InvalidOperationException` if `T` is `char` — use `ExtractString` for UTF-8 text instead.

---

## Memory Management

```csharp
void Reserve(long capacity)            // ensure at least `capacity` bytes of underlying array capacity
void Resize(long size)                 // Reserve(size), then set Size = size (clamping Offset down if needed)
void Remove(long offset, long size)    // remove bytes at offset, shifting remainder left; adjusts Size/Offset
void Clear()                           // Size = 0, Offset = 0 (array is kept, not returned to pool)
void Reset()                           // Clear(); additionally discards & re-rents the array if it exceeds MaxRetainedCapacity
void Attach()                          // detaches current array, leaving Data empty ([]) and unpooled
void Attach(long capacity)             // detaches current array and rents a fresh one of the given (or DefaultCapacity) size
void Attach(byte[] buffer, long offset, long size)  // detaches current array and wraps an existing external array (not pooled)
```

### Shift / Unshift — move read cursor (1-arg overloads)

```csharp
void Shift(long offset)     // advance Offset forward by `offset` (consume bytes); throws if it would exceed Size
void Unshift(long offset)   // retreat Offset backward by `offset` (un-consume bytes); throws if it would go negative
```

### Shift / Unshift — open/close a gap in the data (2-arg overloads)

```csharp
long Shift(long offset, long size)    // shifts bytes at/after `offset` forward by `size`, opening a gap; grows Size
long Unshift(long offset, long size)  // shifts bytes at/after `offset` backward by `size`, closing a gap; shrinks Size
```

These operate on buffer *content* (used internally by `Write(..., force: false)` and `InitStruct<T>`), distinct from the 1-arg `Shift`/`Unshift` which only move the `Offset` read cursor.

---

## Utilities

```csharp
string ExtractString(long offset, long size)  // decode UTF-8 substring relative to Offset
string ToString()                             // decode full unread buffer ([0, Size)) as UTF-8
void Dispose()                                // inherited from PooledObject<Buffer> — returns to pool
static long GetBytesCount<T>(ReadOnlySpan<T> span) where T : unmanaged  // byte count for a generic span
static int SafeToInt32(long value)            // checked long→int cast, throws OverflowException if out of int range
```

---

## Usage

```csharp
// Build a response manually
using var buf = Lucifer.Rent<Buffer>();
buf.Append("HTTP/1.1 200 OK\r\n"u8);
buf.Append("Content-Length: "u8);
buf.AppendUtf8(body.Length);
buf.Append("\r\n\r\n"u8);
buf.Append(body);

session.SendResponseAsync(buf.AsSpan());
```

---

## Remarks

- `Append(ReadOnlySpan<char>)` performs UTF-8 encoding in-place — no intermediate `string` allocation.
- `AppendUtf8(int)` writes the decimal ASCII digits of the integer — not the binary representation. Use `Append(int)` (via the generic `Append<T>`) for binary wire encoding.
- `Reset()` only discards and re-rents the backing array when it exceeds `MaxRetainedCapacity` (256 KB by default); otherwise the array is kept as-is and reused on the next rent, with `Clear()` resetting `Size`/`Offset` to 0.
- `Attach()` with no arguments leaves the buffer with an **empty, unpooled** array (`[]`) — it does not rent a new default-capacity array. Use `Attach(capacity)` if you need a fresh rented array.
- `Offset` and `Size` are `long` for API forward-compatibility, but the underlying array is capped at `MaxBufferCapacity` bytes due to CLR array limits.
- The 1-arg `Shift`/`Unshift` move the read cursor only; the 2-arg `Shift`/`Unshift` mutate buffer content by opening/closing a gap — don't confuse the two.

---