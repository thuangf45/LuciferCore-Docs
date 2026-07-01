# FileCache

**Namespace:** `LuciferCore.Storage`

`FileCache` is an in-memory, thread-safe cache for static files and byte payloads.

It supports:
- key -> byte[] storage
- folder scanning
- optional file watcher auto-reload
- optional entry timeout (TTL)

Often used by server static content loading.

---

## Typical lifecycle

```csharp
AddStaticContent(_staticContentPath);
Cache.Freeze();
```

`Freeze()` locks cache into read-only optimized mode.  
After freeze, write operations are not allowed.

---

## Core methods

### Add

```csharp
public bool Add<T>(ReadOnlySpan<T> key, byte[] value, TimeSpan timeout = default) where T : unmanaged
```

Examples:

```csharp
cache.Add("/custom.json"u8, jsonBytes);
cache.Add("/temp.txt"u8, data, TimeSpan.FromMinutes(5));
```

---

### Find / TryGetValue

```csharp
public (bool found, byte[] value) Find<T>(ReadOnlySpan<T> key) where T : unmanaged
public bool TryGetValue<T>(ReadOnlySpan<T> key, out byte[] value) where T : unmanaged
```

If missing/expired: returns not found and empty value.

---

### Contains / Remove

```csharp
public bool Contains<T>(ReadOnlySpan<T> key) where T : unmanaged
public bool Remove<T>(ReadOnlySpan<T> key) where T : unmanaged
```

---

## Properties

| Property | Type | Meaning |
|---|---|---|
| `IsEmpty` | `bool` | cache empty or not |
| `Size` | `int` | current entry count |

---

## Path scanning

### InsertPath

```csharp
public bool InsertPath(
    string path,
    string prefix = "/",
    string filter = "*.*",
    TimeSpan timeout = default,
    InsertHandler? handler = null)
```

What it does:
- scan folder recursively
- load matching files
- insert into cache using URL prefix
- attach `FileSystemWatcher` for updates

| Parameter | Meaning |
|---|---|
| `path` | source folder |
| `prefix` | URL prefix |
| `filter` | file pattern |
| `timeout` | entry TTL |
| `handler` | optional per-file custom logic |

---

### InsertHandler delegate

```csharp
public delegate bool InsertHandler(FileCache cache, string key, byte[] value, TimeSpan timeout)
```

Return:
- `true` -> keep insert
- `false` -> skip file

---

### FindPath / RemovePath

```csharp
public bool FindPath(string path)
public bool RemovePath(string path)
```

Manage previously registered scan paths/watchers.

---

## Cache management

```csharp
public void Clear()
public void Dispose()
```

- `Clear()` removes entries and stops watchers
- `Dispose()` clears and releases resources

---

## System defaults

```csharp
public void EnsureSystemFiles(InsertHandler handler, string prefix = "/", TimeSpan? timeout = null)
```

Used internally to ensure default framework files exist.

---

## Notes

- Key lookup is UTF-8 span based (fast, low allocation).
- After `Freeze()`, cache is read-only.
- Expired entries are removed lazily on access (no background cleanup thread).
- File changes can auto-refresh cache when path watcher is enabled.