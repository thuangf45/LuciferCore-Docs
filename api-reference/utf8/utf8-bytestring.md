# ByteString & ByteStringComparer

**Namespace:** `LuciferCore.Utf8`

`ByteString` is LuciferCore’s UTF-8 key primitive for low-allocation hot paths.

`ByteStringComparer` is the comparer for hashing/equality (with optional case-insensitive behavior), including span-based alternate lookup support.

---

## ByteString

```csharp
public readonly struct ByteString
```

Key traits:
- immutable readonly struct
- stores byte-array slice view (`offset + size`)
- has precomputed hash
- compares as bytes (no string decode needed)

No public constructor.  
Create with `CopyFrom(...)` or use `ByteString.Empty`.

---

## Factory methods

```csharp
ByteString.CopyFrom(ReadOnlySpan<byte>)
ByteString.CopyFrom(ReadOnlySpan<char>)
ByteString.CopyFrom<T>(ReadOnlySpan<T>) where T : unmanaged
```

Examples:

```csharp
var a = ByteString.CopyFrom("Content-Type"u8);
var b = ByteString.CopyFrom("Authorization".AsSpan());
```

> `CopyFrom` allocates/copies. Good for long-lived keys, not per-request transient lookups.

---

## Important members

| Member | Type | Meaning |
|---|---|---|
| `Size` | `int` | byte length |
| `IsEmpty` | `bool` | empty state |
| `Hash` | `int` | precomputed hash |
| `this[int]` | `byte` | byte index access |
| `AsSpan()` | `Span<byte>` | mutable span view |
| `AsReadOnlySpan()` | `ReadOnlySpan<byte>` | readonly span view |
| `Slice(start, len)` | `ByteString` | copied sub-key |
| `Equals<T>(span, ignoreCase)` | `bool` | span compare |
| `ToString()` | `string` | UTF-8 decode (allocates) |

Implicit conversion:

```csharp
ReadOnlySpan<byte> span = myByteString;
```

---

## Equality behavior

```csharp
byteString.Equals("content-type"u8);                 // case-sensitive
byteString.Equals("Content-Type"u8, ignoreCase: true);
```

Works with `byte` and `char` spans.

---

## ByteStringComparer

```csharp
public sealed class ByteStringComparer :
    IEqualityComparer<ByteString>,
    IAlternateEqualityComparer<ReadOnlySpan<byte>, ByteString>,
    IAlternateEqualityComparer<ReadOnlySpan<char>, ByteString>
```

Constructor:

```csharp
public ByteStringComparer(bool ignoreCase = false)
```

- `ignoreCase = false`: fastest, use stored hash
- `ignoreCase = true`: case-folded hash/equality (ASCII-focused)

---

## Main comparer methods

```csharp
bool Equals(ByteString x, ByteString y)
int GetHashCode(ByteString obj)

ByteString Create(ReadOnlySpan<byte> alt)
bool Equals(ReadOnlySpan<byte> span, ByteString key)
int GetHashCode(ReadOnlySpan<byte> span)

ByteString Create(ReadOnlySpan<char> alt)
bool Equals(ReadOnlySpan<char> span, ByteString key)
int GetHashCode(ReadOnlySpan<char> span)
```

---

## Typical usage

```csharp
var comparer = new ByteStringComparer(ignoreCase: true);
var dict = new Dictionary<ByteString, string>(comparer);

// .NET alternate lookup
var lookup = dict.GetAlternateLookup<ReadOnlySpan<byte>>();
if (lookup.TryGetValue("content-type"u8, out var v))
{
    // no ByteString allocation for lookup key
}
```

---

## Notes

- `ByteString` is value type, but backing byte[] lifetime still matters.
- If backed by pooled memory, do not keep reference after pool return.
- `CopyFrom`/`Slice` create owned copies (safe for long lifetime).
- Avoid `ToString()` on hot path.
- For one-off dictionary lookup, prefer alternate lookup over `CopyFrom(...)`.