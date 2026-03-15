# Buffer

**Namespace:** `LuciferCore.Storage`

`Buffer` is LuciferCore's core pooled byte container. It is a dynamically expandable, append-optimized byte buffer backed by a rented array from `ArrayPool<byte>`. All hot-path types — `PacketModel`, `RequestModel`, `ResponseModel`, `Utf8Builder` — store their data in a `Buffer`.

It extends `PooledObject` and implements `IDisposable`. Obtain instances via `Lucifer.Rent<Buffer>()`.

**Capacity limits:** max 2 GB (`int.MaxValue` bytes). Buffers larger than 256 KB are not retained in the pool on return — they are discarded to avoid long-lived large allocations.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Data` | `byte[]` | The underlying pooled byte array. Both getter and setter call `CheckSafety()` — accessing `Data` on a disposed `Buffer` throws `ObjectDisposedException` |
| `Size` | `long` | Logical size — number of bytes written |
| `Offset` | `long` | Current read position within `Data` |
| `Capacity` | `long` | Physical size of the underlying array |
| `IsEmpty` | `bool` | `true` if `Data` is null or `Size == 0` |
| `IsValid` | `bool` | `true` if the buffer is in a consistent state |
| `MaxSupportedCapacity` | `long` (static) | `2,147,483,647` (2 GB) |

---

## Indexers

```csharp
byte this[long index]        // single byte at index
byte[] this[Range range]     // slice copy via Range syntax
```

---

## Span & Memory Views

All views are zero-copy over the unread region `[Offset, Size)`:

```csharp
Span<byte>          AsSpan()
Span<byte>          AsSpan(long start, long length)
Memory<byte>        AsMemory()
Memory<byte>        AsMemory(long start, long length)
ArraySegment<byte>  AsSegment()
ReadOnlySpan<byte>  Slice(long start, long length)
```

Implicit conversion to `ReadOnlySpan<byte>` is also available:

```csharp
ReadOnlySpan<byte> span = buffer; // implicit
```

---

## Append Methods

All `Append` overloads return the number of bytes written and grow the underlying array automatically if needed.

### Raw bytes & spans

```csharp
long Append(byte value)
long Append(ReadOnlySpan<byte> buffer)
long Append(ReadOnlySpan<char> text)          // UTF-8 encodes chars
long Append<T>(ReadOnlySpan<T> data)          // generic unmanaged span
```

### Binary number overloads (little-endian raw bytes)

```csharp
long Append(int value)    long Append(long value)   long Append(short value)
long Append(uint value)   long Append(ulong value)  long Append(ushort value)
long Append(float value)  long Append(double value) long Append(decimal value)
long Append(bool value)   long Append(char value)   long Append(sbyte value)
```

### UTF-8 text overloads (human-readable digits)

For writing numbers as ASCII text into HTTP responses/requests:

```csharp
long AppendUtf8(int value)    long AppendUtf8(long value)
long AppendUtf8(uint value)   long AppendUtf8(ulong value)
long AppendUtf8(float value)  long AppendUtf8(double value)
long AppendUtf8(decimal value) long AppendUtf8(bool value)
long AppendUtf8(Guid value)   long AppendUtf8(char value)
// + short, ushort, sbyte, byte variants
```

---

## Memory Management

```csharp
void Reserve(long capacity)          // ensure at least `capacity` bytes available
void Resize(long size)               // set logical Size directly
void Remove(long offset, long size)  // remove bytes at offset, shifting remainder left
void Shift(long offset)              // advance Offset forward (consume bytes)
void Unshift(long offset)            // retreat Offset backward (un-consume bytes)
void Reset()                         // clear Size and Offset, return array to pool
void Attach()                        // detach current array and attach a fresh default-capacity one
void Attach(long capacity)           // attach a fresh array of specified capacity
void Attach(byte[] buffer, long offset, long size)  // wrap an existing array
```

---

## Utilities

```csharp
string ExtractString(long offset, long size)  // decode UTF-8 substring
string ToString()                             // decode full buffer as UTF-8
void Dispose()                                // return to pool
static long GetBytesCount<T>(ReadOnlySpan<T> span)  // byte count for a generic span
static int SafeToInt32(long value)            // checked long→int cast
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
- `AppendUtf8(int)` writes the decimal ASCII digits of the integer — not the 4-byte little-endian binary. Use `Append(int)` for binary wire encoding.
- Buffers larger than 256 KB are dropped on `Reset()` rather than pooled, preventing large arrays from being indefinitely retained.
- `Offset` and `Size` are `long` for API forward-compatibility, but the underlying array is capped at `int.MaxValue` bytes due to CLR array limits.
