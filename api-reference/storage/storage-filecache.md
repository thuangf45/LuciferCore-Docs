# FileCache

**Namespace:** `LuciferCore.Storage`

`FileCache` is an in-memory, thread-safe cache for serving static files and byte payloads. It supports keyed byte-array storage, filesystem path scanning with optional `FileSystemWatcher` auto-reload, and time-based expiry.

It is used directly by `WssServer` / `WsServer` via `AddStaticContent()`. Call `Cache.Freeze()` after loading all content to lock the cache into a read-optimized state.

---

## Lifecycle

```csharp
// Inside your [Server] constructor:
AddStaticContent(_staticContentPath); // scan and load all files from path
Cache.Freeze();                        // lock to read-only, optimized lookups
```

`Freeze()` makes the underlying lookup structure immutable. After freezing, `Add`, `Remove`, and `InsertPath` are no longer permitted.

---

## Cache Item Access

### `Add<T>(key, value, timeout)`

```csharp
public bool Add<T>(ReadOnlySpan<T> key, byte[] value, TimeSpan timeout = default)
    where T : unmanaged
```

Inserts a key-value pair. `key` accepts both `byte` (UTF-8) and `char` spans. `timeout` is optional — a zero `TimeSpan` means no expiry.

```csharp
cache.Add("/custom.json"u8, jsonBytes);
cache.Add("/temp.txt"u8, data, TimeSpan.FromMinutes(5));
```

### `Find<T>(key)`

```csharp
public (bool found, byte[] value) Find<T>(ReadOnlySpan<T> key) where T : unmanaged
```

Returns a tuple with the found flag and the cached bytes. Returns `(false, [])` if the key does not exist or has expired.

### `TryGetValue<T>(key, out value)`

```csharp
public bool TryGetValue<T>(ReadOnlySpan<T> key, out byte[] value) where T : unmanaged
```

Standard try-get pattern. Returns `false` and sets `value` to `[]` on miss.

### `Contains<T>(key)` / `Remove<T>(key)`

```csharp
public bool Contains<T>(ReadOnlySpan<T> key) where T : unmanaged
public bool Remove<T>(ReadOnlySpan<T> key) where T : unmanaged
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `IsEmpty` | `bool` | `true` if no entries are cached |
| `Size` | `int` | Number of entries currently in the cache |

---

## Path Management

### `InsertPath()`

```csharp
public bool InsertPath(
    string path,
    string prefix = "/",
    string filter = "*.*",
    TimeSpan timeout = default,
    InsertHandler? handler = null)
```

Scans `path` recursively, loads all matching files (by `filter`), and inserts them under the given `prefix`. Attaches a `FileSystemWatcher` to auto-reload on file change.

| Parameter | Default | Description |
|---|---|---|
| `path` | — | Filesystem directory to scan |
| `prefix` | `"/"` | URL prefix prepended to each file's relative path |
| `filter` | `"*.*"` | Glob pattern for file selection |
| `timeout` | none | Per-entry TTL. Zero means no expiry |
| `handler` | `null` | Optional custom `InsertHandler` delegate called per file |

### `InsertHandler` delegate

```csharp
public delegate bool InsertHandler(FileCache cache, string key, byte[] value, TimeSpan timeout)
```

Called once per file during `InsertPath`. Return `true` to confirm insertion, `false` to skip. Use this to transform or filter file content before caching.

### `FindPath(path)` / `RemovePath(path)`

```csharp
public bool FindPath(string path)
public bool RemovePath(string path)
```

Check or remove all entries associated with a previously scanned path (including the watcher).

---

## Cache Management

```csharp
public void Clear()     // remove all entries and stop all file watchers
public void Dispose()   // clear and release resources
```

---

## System Defaults

```csharp
public void EnsureSystemFiles(InsertHandler handler, string prefix = "/", TimeSpan? timeout = null)
```

Inserts framework-level default files if they are not already cached. Called internally — you do not need to call this manually.

---

## Remarks

- All key lookups use UTF-8 byte-span comparison — no string allocations on hot read paths.
- After `Freeze()`, all write operations are rejected. Only `Find`, `TryGetValue`, and `Contains` remain valid.
- `FileSystemWatcher` is attached per scanned path. File changes trigger automatic cache refresh without requiring a server restart.
- Entries with a non-zero `timeout` are evicted lazily on access — no background timer thread is used.
