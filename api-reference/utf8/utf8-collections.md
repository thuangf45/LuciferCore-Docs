# UTF-8 Collections

**Namespace:** `LuciferCore.Utf8`

LuciferCore now centers on one UTF-8 key collection:

- `Utf8Map<T>`

Older separate types (`IUtf8Store<T>`, `Utf8Dictionary<T>`, `Utf8ConcurrentDictionary<T>`) are removed.

---

## `Utf8Map<T>`

```csharp
public class Utf8Map<T> : IEnumerable<KeyValuePair<ByteString, T>>
```

High-performance UTF-8 key map with:
- span-based lookup
- optional case-insensitive mode
- mutable -> frozen mode transition

---

## Constructor

```csharp
public Utf8Map(bool ignoreCase = false)
```

| Parameter | Default | Meaning |
|---|---|---|
| `ignoreCase` | `false` | ASCII case-insensitive hash/compare |

Internal behavior:
- start mutable with `ConcurrentDictionary<ByteString, T>`
- after `Freeze()` use `Dictionary<ByteString, T>`

---

## Core methods

```csharp
TryAdd<TKey>(ReadOnlySpan<TKey> key, T value)
TryRemove<TKey>(ReadOnlySpan<TKey> key, out T value)
TryGetValue<TKey>(ReadOnlySpan<TKey> key, out T value)
AddOrUpdate<TKey>(ReadOnlySpan<TKey> key, T value)
GetOrAdd<TKey>(ReadOnlySpan<TKey> key, Func<ByteString, T> valueFactory)
```

`TKey` typically:
- `byte` span (`"..."u8`)
- `char` span (`"..."` / `.AsSpan()`)

Lookups are span-based (no key allocation for read path).

---

## Convenience add overloads

```csharp
Add(ReadOnlySpan<char> key, T value)
Add(ReadOnlySpan<byte> key, T value)
Add(string key, string value)
```

`Add(string, string)` works meaningfully for:
- `Utf8Map<ByteString>`
- `Utf8Map<string>`

For other `T`, use typed overloads.

---

## Prefix resolution

```csharp
TryResolvePrefix<TKey>(ReadOnlySpan<TKey> keySpan, TKey slash, out T? value)
    where TKey : unmanaged, IEquatable<TKey>
```

Checks:
1. exact key
2. then parent prefixes split by slash

Example:

```csharp
// registered: /assets
// lookup: /assets/css/main.css -> /assets
```

---

## Indexers

```csharp
T this[ReadOnlySpan<byte> key] { get; set; }
T this[ReadOnlySpan<char> key] { get; set; }
```

- get: throws if missing
- set: add-or-update behavior

---

## Freeze

```csharp
public void Freeze()
```

Switches map to read-optimized dictionary state.

- second call is no-op
- internal concurrent store is released

Important:
- writes after freeze are still possible
- but not safe for concurrent writes (dictionary backend)

Recommended: finish all writes before freeze.

---

## Other members

```csharp
int Count
void Clear()
IEnumerable<KeyValuePair<ByteString, T>> Entries
IEnumerable<ByteString> Keys
IEnumerable<T> Values
```

---

## Example

```csharp
var routes = new Utf8Map<RouteEntry>();
routes.TryAdd("GET:/v1/api/user"u8, entry);
routes.TryAdd("POST:/v1/api/user"u8, entry2);
routes.Freeze();

if (routes.TryGetValue("GET:/v1/api/user"u8, out var route))
    route.SyncInvoker?.Invoke(data, session);
```

Case-insensitive example:

```csharp
var mime = new Utf8Map<ByteString>(ignoreCase: true);
mime.Add(".HTML"u8, ByteString.CopyFrom("text/html"u8));
mime.TryGetValue(".html"u8, out var contentType); // match
```

---

## Notes

- Read lookups use span alternate lookup paths to avoid allocation.
- Stored keys are owned copies (`ByteString.CopyFrom`).
- Comparer is created internally from `ignoreCase`.
- Ideal flow: build -> freeze -> read-heavy usage.