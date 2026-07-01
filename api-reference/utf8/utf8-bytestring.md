# ByteString & ByteStringComparer

**Namespace:** `LuciferCore.Utf8`

`ByteString` is LuciferCore's core zero-allocation UTF-8 string primitive. It is an immutable `readonly struct` that holds a view over a byte array slice with a pre-computed hash — enabling allocation-free string comparisons, dictionary lookups, and span operations on hot paths.

`ByteStringComparer` is the companion comparer used by all UTF-8 collections to hash and compare keys, with optional ASCII case folding. It implements both `IEqualityComparer<ByteString>` and `IAlternateEqualityComparer<TAlternate, ByteString>` for `byte` and `char` spans, enabling allocation-free dictionary lookups with `Dictionary<ByteString, T>.GetAlternateLookup<ReadOnlySpan<byte>>()` / `...<ReadOnlySpan<char>>()` (.NET 9+ alternate lookup feature) without constructing a `ByteString` key.

---

## ByteString

```csharp
public readonly struct ByteString
```

There is no public constructor — instances are created only via the static `CopyFrom` factory methods (or internally by the framework). `ByteString.Empty` is the default/zero-value instance.

### Key Design Points

- **Value type** (`readonly struct`) — passed by value, no heap allocation for the reference itself.
- **Immutable view** — wraps `byte[]` with `offset` + `size`, never copies unless you call `CopyFrom` or `Slice`.
- **Pre-computed hash** — `Hash` is computed once at construction via `Lucifer.ComputeHash<byte>`. Dictionary lookups never rehash (unless using a case-insensitive comparer).
- **Zero-allocation equality** — `Equals()` compares the raw byte spans directly without decoding to `string`.

---

### Static Members

| Member | Description |
|---|---|
| `ByteString.Empty` | Default/empty instance (`_data == null`) |
| `CopyFrom(ReadOnlySpan<byte>)` | Create an owned `ByteString` from a UTF-8 byte span (allocates a new array, copies via `Lucifer.Copy`) |
| `CopyFrom(ReadOnlySpan<char>)` | Create an owned `ByteString` by encoding a `char` span to UTF-8 |
| `CopyFrom<T>(ReadOnlySpan<T>)` where `T : unmanaged` | Generic overload — dispatches to the `byte` or `char` path; any other unmanaged type is reinterpreted as raw bytes |

```csharp
// From UTF-8 literal
var key = ByteString.CopyFrom("Content-Type"u8);

// From char span / string
var key2 = ByteString.CopyFrom("Authorization".AsSpan());

// From generic
var key3 = ByteString.CopyFrom<byte>(someBytes);
```

> `CopyFrom` allocates a new `byte[]` and performs a copy. Use it at startup for constants, or wherever you need a key that outlives the source buffer. On hot lookup paths, prefer `ByteStringComparer`'s alternate-lookup `Equals`/`GetHashCode` overloads over byte/char spans directly, instead of constructing a `ByteString` just to compare it.

---

### Instance Members

| Member | Type | Description |
|---|---|---|
| `Size` | `int` | Length of the view in bytes |
| `IsEmpty` | `bool` | `true` if `Size == 0` or backing array is null |
| `Hash` | `int` | Pre-computed hash code, set at construction |
| `this[int index]` | `byte` | Byte at index (via `AsSpan()[index]`) |
| `AsSpan()` | `Span<byte>` | Mutable span over the view; returns an empty span if the backing array is null |
| `AsReadOnlySpan()` | `ReadOnlySpan<byte>` | Read-only span over the view; same null-safety as `AsSpan()` |
| `Slice(int start, int length)` | `ByteString` | Returns a **new, independently owned** `ByteString` — copies the sliced range into a fresh array, it does not just narrow the existing view |
| `Equals<T>(ReadOnlySpan<T> other, bool ignoreCase = false)` where `T : unmanaged` | `bool` | Zero-allocation equality check against a `byte` or `char` span (any other unmanaged `T` is compared as raw bytes) |
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

`Equals<T>()` accepts `byte` and `char` spans (and any other unmanaged type, compared byte-for-byte) and supports optional ASCII case folding:

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
public sealed class ByteStringComparer :
    IEqualityComparer<ByteString>,
    IAlternateEqualityComparer<ReadOnlySpan<byte>, ByteString>,
    IAlternateEqualityComparer<ReadOnlySpan<char>, ByteString>
```

Implements `IEqualityComparer<ByteString>` for use with .NET collections (`Dictionary`, `ConcurrentDictionary`, `HashSet`), plus the two `IAlternateEqualityComparer<TAlternate, ByteString>` interfaces that let you look up entries in a `Dictionary<ByteString, T>` directly with a `ReadOnlySpan<byte>` or `ReadOnlySpan<char>` key — no `ByteString` allocation needed for the lookup itself. Passed to all UTF-8 collection constructors in LuciferCore.

### Constructor

```csharp
public ByteStringComparer(bool ignoreCase = false)
```

| Parameter | Default | Description |
|---|---|---|
| `ignoreCase` | `false` | When `true`, performs ASCII case-insensitive comparison and hashing |

### Methods

```csharp
// IEqualityComparer<ByteString>
bool Equals(ByteString x, ByteString y)
int  GetHashCode(ByteString obj)

// IAlternateEqualityComparer<ReadOnlySpan<byte>, ByteString>
ByteString Create(ReadOnlySpan<byte> alternate)        // == ByteString.CopyFrom(alternate)
bool       Equals(ReadOnlySpan<byte> span, ByteString key)
int        GetHashCode(ReadOnlySpan<byte> span)

// IAlternateEqualityComparer<ReadOnlySpan<char>, ByteString>
ByteString Create(ReadOnlySpan<char> alternate)         // == ByteString.CopyFrom(alternate)
bool       Equals(ReadOnlySpan<char> span, ByteString other)
int        GetHashCode(ReadOnlySpan<char> span)
```

When `ignoreCase` is `false`, `GetHashCode(ByteString obj)` returns the pre-computed `obj.Hash` directly — no recomputation. When `ignoreCase` is `true`, the hash is recomputed with case folding applied (`Lucifer.ComputeHash<byte>(obj.AsSpan(), true)`). The span-based `GetHashCode` overloads always compute the hash fresh via `Lucifer.ComputeHash`, respecting `ignoreCase`.

### Usage

```csharp
// Case-sensitive comparer (default)
var comparer = new ByteStringComparer();

// Case-insensitive comparer — used by StorageData.MimeTable, FileCache, etc.
var ciComparer = new ByteStringComparer(ignoreCase: true);

var dict = new Dictionary<ByteString, string>(ciComparer);

// Allocation-free lookup via alternate comparer (.NET 9+ GetAlternateLookup)
var lookup = dict.GetAlternateLookup<ReadOnlySpan<byte>>();
if (lookup.TryGetValue("content-type"u8, out var value)) { /* ... */ }
```

---

## Remarks

- `ByteString` is a `readonly struct` — storing it in a field or returning it from a method costs no heap allocation.
- The backing `byte[]` is **not owned** by a view-style `ByteString` — it is the same array passed into the (internal) constructor. If the underlying array is a pooled `Buffer`'s array, do not use the `ByteString` after the buffer is returned to the pool. `CopyFrom` and `Slice` always allocate their own independent array, so `ByteString`s produced by them are safe to keep beyond the lifetime of the source.
- There's no public way to construct a `ByteString` directly over an arbitrary array/offset/size — only `CopyFrom` (copying) is publicly available; the offset/size constructor is `internal`.
- `ToString()` allocates a managed `string` — call it only for debugging and logging, never in hot dispatch paths.
- Prefer `ByteStringComparer`'s `IAlternateEqualityComparer` overloads (`Equals`/`GetHashCode` taking spans, used via `GetAlternateLookup`) over manually calling `ByteString.CopyFrom` just to perform a single dictionary lookup — it avoids the allocation entirely.

---