# Writing a Custom Service

This guide walks through implementing a service by inheriting `ServiceBase`. The built-in system services (`RateLimitService`, `SessionService`, `FinanceService`) are used throughout as real-world reference examples.

---

## Registering with the System

Decorate the class with `[Service]` to register it with LuciferCore. The `name` string is the unique identifier used in console commands and log output.

```csharp
using LuciferCore.Attributes;

[Service("MyService")]
internal sealed class MyService : ServiceBase
{
    // ...
}
```

**Namespace:** `LuciferCore.Attributes`

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Unique identifier for this service. Used internally as a `ByteString` key for fast lookup. |

`Lucifer.Run()` auto-discovers all `[Service]`-decorated classes, instantiates them, and calls `Start()` — no manual registration needed. The attribute is `AllowMultiple = false` — one name per class. Without it the service will not be discovered or registered by the system.

---

## Minimal Implementation

Every service needs the `[Service]` attribute and exactly one override — `Update()`:

```csharp
[Service("MyService")]
internal sealed class MyService : ServiceBase
{
    protected override void Setup()
    {
        Interval = TimeSpan.FromMinutes(30);
    }

    protected override void Update()
    {
        // your periodic logic
        Workload = /* current item count */;
    }
}
```

---

## Setting the Interval

Set `Interval` inside `Setup()` — before the timer is registered. All built-in services follow this pattern:

```csharp
// RateLimitService — evicts stale rate state entries once per hour
protected override void Setup() => Interval = TimeSpan.FromHours(1);

// SessionService — heavy user/session maintenance, runs every 6 hours
protected override void Setup() => Interval = TimeSpan.FromHours(6);

// FinanceService — recalculates adaptive delays for all managers every 5 seconds
protected override void Setup() => Interval = TimeSpan.FromSeconds(5);
```

Choose the interval based on how stale the data can be before it becomes a problem. Services are not suited for sub-second work — use `ManagerBase` for that.

---

## Reporting Workload

Set `Workload` at the end of each `Update()` to reflect the current item count. This value is readable from outside the service for monitoring:

```csharp
// RateLimitService
protected override void Update()
{
    Cleanup(_batchSize);
    Workload = _session.Count;
}

// SessionService
protected override void Update()
{
    Maintenance();
    Workload = _sessions.Count;
}
```

---

## Processing in Batches

Always cap work per `Update()` cycle with a `_batchSize` field. The timer callback runs on a **thread pool thread** — holding it too long blocks other thread pool work across the entire process:

```csharp
private readonly int _batchSize = 5_000;

protected override void Update()
{
    var removed = Cleanup(_batchSize);
    Workload = _entries.Count;
}

private int Cleanup(int batchSize)
{
    var removed = 0;
    foreach (var kv in _entries)
    {
        if (removed >= batchSize) break;
        // ... remove expired entries
        removed++;
    }
    return removed;
}
```

**Real example — RateLimitService:**

```csharp
private readonly int _batchSize = 5000;

private int Cleanup(int batchSize)
{
    var now = Lucifer.Time;
    var ttl = _entryTTL;
    var removed = 0;

    foreach (var kv in _session)
    {
        if (removed >= batchSize) break;
        if (now - kv.Value.LastAccessSeconds > ttl)
        {
            if (_session.TryRemove(kv.Key, out var state))
            {
                Lucifer.Return(state); // return to pool, not GC
                removed++;
            }
        }
    }
    return removed;
}
```

---

## Using Pooled Objects in Cleanup

When evicting entries that extend `PooledObject`, always return them to the pool via `Lucifer.Return()` rather than letting the GC collect them:

```csharp
// RateLimitService — RateState extends PooledObject
if (_session.TryRemove(kv.Key, out var state))
{
    lock (state) state.IsRemoved = true;
    Lucifer.Return(state); // ← pool, not GC
    removed++;
}

// SessionService — SessionEntry extends PooledObject
if (_sessions.TryRemove(sessionId, out var entry))
{
    lock (entry)
    {
        entry.IsRemoved = true;
        Lucifer.Return(entry);
    }
}
```

---

## Resetting State on Restart

Override `Cleanup()` to reset mutable state when the service is restarted mid-run. `Setup()` is **not** called again on restart — only `Cleanup()` is:

```csharp
protected override void Cleanup()
{
    _entries.Clear();
    Workload = 0;
}
```

---

## Cleanup on Dispose

Override `Dispose(bool)` to release resources. Always call `base.Dispose(disposing)` at the end:

```csharp
// Simple clear
protected override void Dispose(bool disposing)
{
    _entries.Clear();
    base.Dispose(disposing);
}

// With pooled objects — return to pool before clearing
protected override void Dispose(bool disposing)
{
    foreach (var kv in _entries)
        Lucifer.Return(kv.Value);

    _entries.Clear();
    base.Dispose(disposing);
}
```

---

## Complete Example

A service that runs every 30 minutes to evict expired cache entries, using pooled objects and batch capping:

```csharp
[Service("CacheCleanup")]
internal sealed class CacheCleanupService : ServiceBase
{
    private readonly int _batchSize = 5_000;
    private readonly double _ttlSeconds = TimeSpan.FromMinutes(10).TotalSeconds;
    private readonly ConcurrentDictionary<long, CacheEntry> _entries = new();

    protected override void Setup()
    {
        Interval = TimeSpan.FromMinutes(30);
    }

    protected override void Update()
    {
        var removed = Cleanup(_batchSize);
        Workload = _entries.Count;

        if (removed > 0)
            Lucifer.Log(this, $"Evicted {removed} expired entries", LogLevel.DEBUG);
    }

    private int Cleanup(int batchSize)
    {
        var now = Lucifer.Time;
        var removed = 0;

        foreach (var kv in _entries)
        {
            if (removed >= batchSize) break;

            if (now - kv.Value.LastAccess > _ttlSeconds)
            {
                if (_entries.TryRemove(kv.Key, out var entry))
                {
                    Lucifer.Return(entry);
                    removed++;
                }
            }
        }

        return removed;
    }

    public void Add(long key, CacheEntry entry) => _entries.TryAdd(key, entry);

    protected override void Cleanup()
    {
        foreach (var kv in _entries) Lucifer.Return(kv.Value);
        _entries.Clear();
        Workload = 0;
    }

    protected override void Dispose(bool disposing)
    {
        foreach (var kv in _entries) Lucifer.Return(kv.Value);
        _entries.Clear();
        base.Dispose(disposing);
    }
}
```
