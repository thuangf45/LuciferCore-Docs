# Service System

A **Service** is a background task that runs on a fixed timer — ideal for recurring maintenance work such as session cleanup, cache eviction, and data synchronization. Unlike a Manager, a Service does not hold a dedicated thread between ticks; it borrows a thread pool thread for the duration of each `Update()` call and returns it immediately.

---

## The `[Service]` Attribute

```csharp
[Service("CleanupService")]
internal sealed class CleanupService : ServiceBase { ... }
```

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | A human-readable identifier for this service |

LuciferCore auto-discovers and starts all `[Service]`-decorated classes when `Lucifer.Run()` is called — identical to how `[Manager]` works.

---

## Lifecycle Methods

Extend `ServiceBase` and override these methods:

### `Setup()`

Called once when the service starts. Use it to configure the timer interval.

```csharp
protected override void Setup()
{
    Interval = TimeSpan.FromMinutes(30); // how often Update() is called
}
```

| Property | Description |
|---|---|
| `Interval` | How often `Update()` is called. Set this in `Setup()` before the timer starts. |

### `Update()`

Called repeatedly on the configured interval. This is where your recurring logic lives. Runs on a thread pool thread — keep it non-blocking.

```csharp
protected override void Update()
{
    // periodic maintenance logic
    Workload = _entries.Count;
}
```

### `Dispose(bool disposing)`

Called when the service is stopped or the host shuts down. Always call `base.Dispose(disposing)`.

```csharp
protected override void Dispose(bool disposing)
{
    _entries.Clear();
    base.Dispose(disposing);
}
```

---

## Full Example

```csharp
[Service("CleanupService")]
internal sealed class CleanupService : ServiceBase
{
    private readonly int _batchSize = 5_000;
    private readonly ConcurrentDictionary<long, CacheEntry> _entries = new();

    protected override void Setup()
    {
        Interval = TimeSpan.FromMinutes(30);
    }

    protected override void Update()
    {
        var removed = Cleanup(_batchSize);
        Workload = _entries.Count;
    }

    private int Cleanup(int batchSize)
    {
        var now = Lucifer.Time;
        var removed = 0;

        foreach (var kv in _entries)
        {
            if (removed >= batchSize) break;
            if (now - kv.Value.LastAccess > 600) // 10 min TTL
            {
                if (_entries.TryRemove(kv.Key, out _))
                    removed++;
            }
        }

        return removed;
    }

    protected override void Dispose(bool disposing)
    {
        _entries.Clear();
        base.Dispose(disposing);
    }
}
```

---

## Manager vs Service

| | Manager | Service |
|---|---|---|
| Thread model | Dedicated OS thread, always alive | Thread pool thread, borrowed per tick |
| Scheduling | Adaptive — sleep shrinks under load | Fixed interval |
| Best for | High-throughput queues, dispatch loops | Periodic cleanup, maintenance, sync |
| Wakeable | Yes — `NotifyWork()` | No |
| Stall detection | Yes — `IsStalled` | No |

Use a **Service** when the task runs infrequently and latency of ±`Interval` is acceptable. Use a **Manager** when the task reacts to unpredictable bursts or needs sub-millisecond responsiveness.

---

## Managing from the Console

Once running, services can be controlled via built-in console commands:

```
/start services
/stop services
/restart services
```

See [Entry Point](../getting-started/entry-point.md) for the full command reference.
