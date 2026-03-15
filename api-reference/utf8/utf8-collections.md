# UTF-8 Collections

**Namespace:** `LuciferCore.Utf8`

LuciferCore provides a family of UTF-8 key collections built around `ByteString` keys and zero-allocation span lookups. All collections implement `IUtf8Store<T>`.

---

## IUtf8Store\<T\>

```csharp
public interface IUtf8Store<T>
```

The common contract for all UTF-8 key stores. Provides the standard dictionary operations using `ByteString` keys, plus enumerable access to entries, keys, and values.

| Member | Description |
|---|---|
| `Count` | Number of entries |
| `this[ByteString key]` | Get or set by key |
| `TryAdd(key, value)` | Add if key does not exist |
| `TryRemove(key, out value)` | Remove and retrieve |
| `TryGetValue(key, out value)` | Try get by key |
| `ContainsKey(key)` | Existence check |
| `Set(key, value)` | Insert or overwrite |
| `Clear()` | Remove all entries |
| `Entries` | All key-value pairs |
| `Keys` | All keys |
| `Values` | All values |

---

## Utf8Map\<T\>

```csharp
public class Utf8Map<T> : IEnumerable<KeyValuePair<ByteString, T>>
```

The primary high-performance UTF-8 map used throughout LuciferCore — for the route table (`HandlerBase.Routes`), the MIME table (`StorageData.MimeTable`), and static file cache lookups. It combines a two-phase lookup (hash bucket → `ByteString` equality) with span-based key matching and an optional `Freeze()` transition for read-only optimization.

### Constructor

```csharp
public Utf8Map(bool ignoreCase = false)
```

| Parameter | Default | Description |
|---|---|---|
| `ignoreCase` | `false` | When `true`, all lookups and hashing are ASCII case-insensitive |

### Core Operations

All lookup and mutation methods accept generic `ReadOnlySpan<TKey>` — no `ByteString` allocation required for reads:

```csharp
bool TryAdd<TKey>(ReadOnlySpan<TKey> key, T value)      where TKey : unmanaged
bool TryRemove<TKey>(ReadOnlySpan<TKey> key, out T value) where TKey : unmanaged
bool TryGetValue<TKey>(ReadOnlySpan<TKey> key, out T value) where TKey : unmanaged
bool AddOrUpdate<TKey>(ReadOnlySpan<TKey> key, T value)  where TKey : unmanaged
T    GetOrAdd<TKey>(ReadOnlySpan<TKey> key, Func<ByteString, T> factory) where TKey : unmanaged
```

`TKey` accepts `byte` (UTF-8 bytes) or `char` (Unicode chars, UTF-8 encoded on comparison).

### Convenience Overloads

```csharp
void Add(ReadOnlySpan<byte> key, T value)
void Add(ReadOnlySpan<char> key, T value)
void Add(string key, string value)          // collection initializer support for Utf8Map<ByteString>
```

### Prefix Resolution

```csharp
bool TryResolvePrefix<TKey>(ReadOnlySpan<TKey> keySpan, TKey slash, out T? value)
    where TKey : unmanaged, IEquatable<TKey>
```

Attempts an exact match first, then walks backwards through `/`-delimited path segments until a registered prefix is found. Used internally for static file routing.

```csharp
// Registered: "/assets"
// Lookup: "/assets/css/main.css" → matches "/assets"
map.TryResolvePrefix("/assets/css/main.css"u8, (byte)'/', out var handler);
```

### Indexers

```csharp
T this[ByteString key] { get; set; }   // direct ByteString key
T this[string key]     { get; set; }   // string key (span comparison, no allocation)
```

### Freeze

```csharp
public void Freeze()
```

Transitions the map from mutable (`Utf8ConcurrentDictionary`) to read-only (`Utf8Dictionary`), and converts the hash bucket index from `ConcurrentDictionary` to a plain `Dictionary`. After freezing, lookups are faster with no concurrent write overhead. Write operations are no-ops after freeze.

Called automatically by `HandlerBase.Optimize()` after all routes are registered, and by `Cache.Freeze()` after static content is loaded.

### Other Members

```csharp
int Count { get; }
void Clear()
IEnumerable<KeyValuePair<ByteString, T>> Entries
IEnumerable<ByteString> Keys
IEnumerable<T> Values
```

### Usage

```csharp
// Build phase
var routes = new Utf8Map<RouteEntry>();
routes.TryAdd("GET:/v1/api/user"u8, entry);
routes.TryAdd("POST:/v1/api/user"u8, entry2);
routes.Freeze(); // lock for read-only

// Lookup phase (zero-allocation)
if (routes.TryGetValue("GET:/v1/api/user"u8, out var route))
    route.SyncInvoker?.Invoke(handler, data, session);

// Case-insensitive map
var mime = new Utf8Map<ByteString>(ignoreCase: true);
mime.Add(".HTML"u8, ByteString.CopyFrom("text/html"u8));
mime.TryGetValue(".html"u8, out var contentType); // matches
```

---

## Utf8Dictionary\<T\>

```csharp
public sealed class Utf8Dictionary<T> : IEnumerable<KeyValuePair<ByteString, T>>, IUtf8Store<T>
```

A single-threaded `Dictionary<ByteString, T>` wrapper. Used by `Utf8Map` internally after `Freeze()` for fast, contention-free read-only lookups.

**Not thread-safe for concurrent writes.** Use `Utf8ConcurrentDictionary<T>` if writes may occur from multiple threads.

### Constructor

```csharp
public Utf8Dictionary(ByteStringComparer comparer)
```

---

## Utf8ConcurrentDictionary\<T\>

```csharp
public class Utf8ConcurrentDictionary<T> : IEnumerable<KeyValuePair<ByteString, T>>, IUtf8Store<T>
```

A `ConcurrentDictionary<ByteString, T>` wrapper. Used by `Utf8Map` during its mutable build phase (before `Freeze()`). Thread-safe for concurrent reads and writes.

### Constructor

```csharp
public Utf8ConcurrentDictionary(ByteStringComparer comparer)
```

---

## Choosing the Right Collection

| Scenario | Recommended Type |
|---|---|
| Route table, MIME table, any read-heavy map | `Utf8Map<T>` with `Freeze()` |
| Single-threaded build + read | `Utf8Map<T>` (manages transition automatically) |
| Persistent concurrent writes required | `Utf8ConcurrentDictionary<T>` directly |
| Single-threaded read-only after build | `Utf8Dictionary<T>` directly |

In most cases, use `Utf8Map<T>` — it handles the mutable → frozen transition automatically and provides the span-based API.

---

## Remarks

- All `TryGetValue` span overloads on `Utf8Map` compute a hash and scan the matching bucket — no `ByteString` is allocated for the lookup key.
- `ByteString` keys stored in the map are owned copies (created via `ByteString.CopyFrom`). The map does not hold references to caller-owned spans.
- `Utf8Dictionary` and `Utf8ConcurrentDictionary` require a `ByteStringComparer` instance — always pass one explicitly to control case sensitivity.
- After `Freeze()`, any attempt to write to a `Utf8Map` is silently ignored. Design your startup flow to complete all writes before calling `Freeze()`.
