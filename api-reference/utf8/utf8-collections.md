# UTF-8 Collections

**Namespace:** `LuciferCore.Utf8`

LuciferCore provides a single high-performance UTF-8 key collection, `Utf8Map<T>`, built around `ByteString` keys and zero-allocation span lookups. It is used throughout LuciferCore — for the route table (`HandlerBase.Routes`), the MIME table (`StorageData.MimeTable`), and static file cache lookups.

> **Note:** Earlier versions of LuciferCore exposed separate `IUtf8Store<T>`, `Utf8Dictionary<T>`, and `Utf8ConcurrentDictionary<T>` types. These have been removed. `Utf8Map<T>` now manages its own internal `Dictionary<ByteString, T>` (frozen/read-only state) and `ConcurrentDictionary<ByteString, T>` (mutable/build state) directly, switching between them via `Freeze()`.

---

## Utf8Map\<T\>

```csharp
public class Utf8Map<T> : IEnumerable<KeyValuePair<ByteString, T>>
```

A thread-safe map with efficient lookups and optional case-insensitivity. It combines a two-phase lookup (hash bucket → `ByteString` equality) with span-based key matching via `Dictionary<TKey, TValue>.AlternateLookup<TAlternateKey>`, and an optional `Freeze()` transition for read-only optimization.

### Constructor

```csharp
public Utf8Map(bool ignoreCase = false)
```

| Parameter | Default | Description |
|---|---|---|
| `ignoreCase` | `false` | When `true`, all lookups and hashing are ASCII case-insensitive |

Internally, a `Utf8Map<T>` starts in a **mutable** state backed by `ConcurrentDictionary<ByteString, T>` (thread-safe for concurrent reads/writes). After calling `Freeze()`, it transitions to a **read-only** state backed by a plain `Dictionary<ByteString, T>` for faster, contention-free lookups.

### Core Operations

All lookup and mutation methods accept generic `ReadOnlySpan<TKey>` — no `ByteString` allocation required for reads, since they use `GetAlternateLookup<TAlternateKey>` under the hood:

```csharp
bool TryAdd<TKey>(ReadOnlySpan<TKey> key, T value)        where TKey : unmanaged
bool TryRemove<TKey>(ReadOnlySpan<TKey> key, out T value) where TKey : unmanaged
bool TryGetValue<TKey>(ReadOnlySpan<TKey> key, out T value) where TKey : unmanaged
bool AddOrUpdate<TKey>(ReadOnlySpan<TKey> key, T value)    where TKey : unmanaged
T    GetOrAdd<TKey>(ReadOnlySpan<TKey> key, Func<ByteString, T> valueFactory) where TKey : unmanaged
```

`TKey` accepts `byte` (UTF-8 bytes) or `char` (Unicode chars, UTF-8 encoded on comparison). For any other unmanaged `TKey`, the span is reinterpreted as raw bytes via `MemoryMarshal.AsBytes`.

`GetOrAdd` looks up the key first; if missing, it builds a `ByteString` from the key span, invokes `valueFactory` to create the value, and adds it via `AddOrUpdate`.

### Convenience Overloads

```csharp
void Add(ReadOnlySpan<char> key, T value)
void Add(ReadOnlySpan<byte> key, T value)
void Add(string key, string value)
```

`Add(string, string)` supports collection-initializer syntax and behaves differently depending on `T`:

- For `Utf8Map<ByteString>`, the string value is converted via `ByteString.CopyFrom`.
- For `Utf8Map<string>`, the string value is used directly.
- For any other `T`, this overload throws `InvalidOperationException`, since there's no way to convert a `string` to an arbitrary `T`. Use the typed `Add(ReadOnlySpan<char>, T)` overload instead.

### Prefix Resolution

```csharp
bool TryResolvePrefix<TKey>(ReadOnlySpan<TKey> keySpan, TKey slash, out T? value)
    where TKey : unmanaged, IEquatable<TKey>
```

Attempts an exact match first, then walks backwards through `slash`-delimited segments (using `Lucifer.LastIndexOf`) until a registered prefix is found. Used internally for static file routing.

```csharp
// Registered: "/assets"
// Lookup: "/assets/css/main.css" → matches "/assets"
map.TryResolvePrefix("/assets/css/main.css"u8, (byte)'/', out var handler);
```

### Indexers

```csharp
T this[ReadOnlySpan<byte> key] { get; set; }   // byte span key
T this[ReadOnlySpan<char> key] { get; set; }   // char span key
```

Getters throw `KeyNotFoundException` if the key is absent. Setters call `AddOrUpdate`, so assigning to an existing key overwrites it and assigning to a new key inserts it.

> There is no indexer that takes a `ByteString` or `string` directly — pass a span (e.g. `map["key"]` works via implicit `ReadOnlySpan<char>` conversion, or use `"key"u8` for a byte span).

### Freeze

```csharp
public void Freeze()
```

Transitions the map from mutable (internal `ConcurrentDictionary<ByteString, T>`) to read-only (internal `Dictionary<ByteString, T>`), copying all entries across using the map's `ByteStringComparer`. Calling `Freeze()` more than once is a no-op (it returns immediately if already frozen). After freezing, `_concurrent` is set to `null` to free memory.

Called automatically by `HandlerBase.Optimize()` after all routes are registered, and by `Cache.Freeze()` after static content is loaded.

> Unlike before, write methods (`Add`, `TryAdd`, `AddOrUpdate`, indexer setters) are **not** silently ignored after `Freeze()` — they continue to operate against the frozen internal `Dictionary<ByteString, T>` (which is not thread-safe for concurrent writes). Avoid writing after `Freeze()` unless you're certain access is single-threaded.

### Other Members

```csharp
int Count { get; }
void Clear()
IEnumerable<KeyValuePair<ByteString, T>> Entries
IEnumerable<ByteString> Keys
IEnumerable<T> Values
```

`Clear()` clears whichever internal store is currently active (`_dict` or `_concurrent`).

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

// Utf8Map<string> also supports Add(string, string) directly
var labels = new Utf8Map<string>();
labels.Add("status", "ok");
```

---

## Remarks

- All `TryGetValue` span overloads compute a hash and scan the matching bucket via `AlternateLookup<TAlternateKey>` — no `ByteString` is allocated for the lookup key.
- `ByteString` keys stored in the map are owned copies (created via `ByteString.CopyFrom`). The map does not hold references to caller-owned spans.
- `Utf8Map<T>` constructs its own `ByteStringComparer` internally from the `ignoreCase` flag — there's no need (or way) to pass one in externally.
- Design your startup flow to complete all writes before calling `Freeze()`, since post-freeze writes are no longer blocked, only operating on a non-concurrent-safe store.

---