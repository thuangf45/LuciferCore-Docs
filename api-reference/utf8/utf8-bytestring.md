# ByteString & ByteStringComparer

**Namespace:** `LuciferCore.Utf8`

`ByteString` is LuciferCore's core zero-allocation UTF-8 string primitive. It is an immutable `readonly struct` that holds a view over a byte array slice with a pre-computed hash — enabling allocation-free string comparisons, dictionary lookups, and span operations on hot paths.

`ByteStringComparer` is the companion `IEqualityComparer<ByteString>` used by all UTF-8 collections to hash and compare keys, with optional ASCII case folding.

---

## ByteString

```csharp
public readonly struct ByteString
```

### Key Design Points

- **Value type** (`readonly struct`) — passed by value, no heap allocation for the reference itself.
- **Immutable view** — wraps `byte[]` with `offset` + `size`, never copies unless you call `CopyFrom`.
- **Pre-computed hash** — `Hash` is computed once at construction using SIMD-accelerated hashing. Dictionary lookups never rehash.
- **Zero-allocation equality** — `Equals()` compares the raw byte spans directly without decoding to `string`.

---

### Static Members

| Member | Description |
|---|---|
| `ByteString.Empty` | Singleton empty instance |
| `CopyFrom(ReadOnlySpan<byte>)` | Create an owned `ByteString` from a UTF-8 byte span (SIMD copy) |
| `CopyFrom(ReadOnlySpan<char>)` | Create an owned `ByteString` by encoding a `char` span to UTF-8 |
| `CopyFrom<T>(ReadOnlySpan<T>)` | Generic overload — dispatches to `byte` or `char` path |

```csharp
// From UTF-8 literal
var key = ByteString.CopyFrom("Content-Type"u8);

// From char span / string
var key2 = ByteString.CopyFrom("Authorization".AsSpan());

// From generic
var key3 = ByteString.CopyFrom<byte>(someBytes);
```

> `CopyFrom` allocates a new `byte[]` and performs a SIMD copy. Use it at startup for constants. On hot paths, use span overloads on `Utf8Map` / `Utf8Builder` directly.

---

### Instance Members

| Member | Type | Description |
|---|---|---|
| `Size` | `int` | Length of the view in bytes |
| `IsEmpty` | `bool` | `true` if `Size == 0` or backing array is null |
| `Hash` | `int` | Pre-computed hash code |
| `this[int index]` | `byte` | Byte at index |
| `AsSpan()` | `Span<byte>` | Mutable span over the view |
| `AsReadOnlySpan()` | `ReadOnlySpan<byte>` | Read-only span over the view |
| `Slice(int start, int length)` | `ByteString` | New owned `ByteString` over a sub-range |
| `Equals<T>(ReadOnlySpan<T>, bool ignoreCase)` | `bool` | Zero-allocation equality check against a `byte` or `char` span |
| `ToString()` | `string` | Decode as UTF-8 string (allocates — avoid on hot paths) |

---

### Implicit Conversion

```csharp
public static implicit operator ReadOnlySpan<byte>(ByteString b)
```

Allows `ByteString` to be passed directly anywhere a `ReadOnlySpan<byte>` is expected:

```csharp
builder.Append(someByteString); // implicit conversion
```

---

### Equality

`Equals<T>()` accepts `byte` and `char` spans and supports optional ASCII case folding:

```csharp
// Case-sensitive (default)
bool match = byteString.Equals("content-type"u8);

// Case-insensitive
bool match2 = byteString.Equals("Content-Type"u8, ignoreCase: true);
```

Comparisons against `char` spans perform UTF-8 byte comparison directly — no intermediate string allocation.

---

## ByteStringComparer

```csharp
public sealed class ByteStringComparer : IEqualityComparer<ByteString>
```

Implements `IEqualityComparer<ByteString>` for use with .NET collections (`Dictionary`, `ConcurrentDictionary`, `HashSet`). Passed to all UTF-8 collection constructors in LuciferCore.

### Constructor

```csharp
public ByteStringComparer(bool ignoreCase = false)
```

| Parameter | Default | Description |
|---|---|---|
| `ignoreCase` | `false` | When `true`, performs ASCII case-insensitive comparison and hashing |

### Methods

```csharp
bool Equals(ByteString x, ByteString y)
int  GetHashCode(ByteString obj)
```

When `ignoreCase` is `false`, `GetHashCode` returns the pre-computed `obj.Hash` directly — no recomputation. When `ignoreCase` is `true`, the hash is recomputed with case folding applied.

### Usage

```csharp
// Case-sensitive comparer (default)
var comparer = new ByteStringComparer();

// Case-insensitive comparer — used by StorageData.MimeTable, FileCache, etc.
var ciComparer = new ByteStringComparer(ignoreCase: true);

var dict = new Dictionary<ByteString, string>(ciComparer);
```

---

## Remarks

- `ByteString` is a `readonly struct` — storing it in a field or returning it from a method costs no heap allocation.
- The backing `byte[]` is **not owned** by the struct — it is a view. If the underlying array is a pooled `Buffer`, do not use the `ByteString` after the buffer is returned to the pool.
- Use `CopyFrom` when you need a persistent, independently owned `ByteString` (e.g. for dictionary keys or static constants).
- `ToString()` allocates a managed `string` — call it only for debugging and logging, never in hot dispatch paths.
