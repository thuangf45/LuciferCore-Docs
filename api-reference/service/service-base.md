# ServiceBase

**Namespace:** `LuciferCore.Service`

Abstract base class for all LuciferCore services. Implements a timer-driven background task — the OS fires an I/O clock interrupt at the configured `Interval`, borrows a thread pool thread, runs `Update()`, then returns it. No thread is held between ticks.

```csharp
public abstract class ServiceBase : IDisposable
```

---

## When to Use

`ServiceBase` is designed for **lightweight, predictable tasks** — session cleanup, rate limit eviction, adaptive delay recalculation — that run on a fixed schedule where ±`Interval` latency is acceptable. Zero fixed thread cost between ticks.

For heavy, continuously running tasks that react to unpredictable bursts or need adaptive latency, use [`ManagerBase`](manager-base.md) instead — it owns a dedicated thread that stays alive and adapts its sleep interval to load.

---

## Thread Model

```
Start()
    → Setup()                            — one-time initialization
    → Lucifer.SetInterval(Run, Interval) — OS timer registered
    → timer fires every Interval:
        └─ Run() → Update()              — thread pool thread, borrowed and returned
Stop()
    → timer.Change(Infinite, Infinite)   — timer suspended, no thread held
Restart()
    → Stop() → Cleanup() → timer.Change(Interval, Interval)
Dispose()
    → timer.Dispose() → Dispose(bool)
```

`Stop()` suspends the timer without disposing it — `Start()` resumes immediately without re-calling `Setup()`. `Restart()` runs `Cleanup()` then resumes the timer, useful for resetting state mid-run.

---

## Lifecycle API

```csharp
void Start()
void Stop()
void Dispose()
```

| Method | Description |
|---|---|
| `Start()` | Calls `Setup()` on first call, then registers the OS timer. Subsequent calls after `Stop()` resume the timer — no re-setup. No-op if already running or disposed. |
| `Stop()` | Suspends the timer. Thread pool thread is returned immediately. Calls `OnStopped()`. |
| `Dispose()` | Disposes the timer and calls `Dispose(bool)`. Asserts single-call — double-dispose is a bug. |

---

## State & Workload

| Member | Type | Description |
|---|---|---|
| `Workload` | `int` | Current load metric — e.g., session count, queue depth. Updated by the derived class in `Update()`. Readable from any thread for monitoring. |

Unlike `ManagerBase`, there is no `IsStalled`, no `NotifyWork()`, and no adaptive delay. Services run strictly on their fixed `Interval`.

---

## Delay & Scheduling

| Property | Default | Description |
|---|---|---|
| `Interval` | `500` ms | How often `Update()` is called. Set this in `Setup()`. |

There is no adaptive budget — the interval is fixed for the lifetime of the service. If you need adaptive scheduling, use `ManagerBase`.

---

## Override Points

| Method | Required | Description |
|---|---|---|
| `Update()` | **Yes** | Main work body. Called every `Interval` on a thread pool thread. |
| `Setup()` | No | One-time init when `Start()` is first called. Set `Interval` here. |
| `Cleanup()` | No | Called during `Restart()` before the timer resumes. Reset state here. |
| `Dispose(bool disposing)` | No | Override for custom disposal. Always call `base.Dispose(disposing)`. |
| `OnStarted()` | No | Default: logs `"Started"`. Override to add custom start logic. |
| `OnStopped()` | No | Default: logs `"Stopped"`. Override to add custom stop logic. |

---

## Built-in Services

LuciferCore ships three internal services as reference implementations:

### RateLimitService

Runs every **1 hour** to evict expired `RateState` entries from the session and global maps. TTL is 10 minutes per entry. `RateState` objects are pooled — eviction returns them to `Lucifer`'s pool rather than GC.

```csharp
protected override void Setup() => Interval = TimeSpan.FromHours(1);
protected override void Update() { Cleanup(_batchSize); Workload = _session.Count; }
```

### SessionService

Runs every **6 hours** for session and user maintenance. Users expire after 14 days (2 hours for `Admin`). Sessions expire after 10 minutes of inactivity. `Admin` sessions enforce single-session policy — existing sessions are force-disconnected on new login.

```csharp
protected override void Setup() => Interval = TimeSpan.FromHours(6);
protected override void Update() { Maintenance(); Workload = _sessions.Count; }
```

### FinanceService

Runs every **5 seconds** to recalculate adaptive delays for all registered `ManagerBase` instances. Uses an EMA (Exponential Moving Average) of each manager's `Workload` against its registered threshold to scale `TimeDelay` between `MinDelay` and `MaxDelay`. When workload is zero for enough cycles, sets `TimeDelay = Timeout.Infinite` to park the manager thread until the next `NotifyWork()`.

```csharp
protected override void Setup() => Interval = TimeSpan.FromSeconds(5);
protected override void Update() { /* recalculate adaptive delays */ Workload = totalWorkload; }
```

---

## Complete Example

A service that runs every hour to clean up stale entries, with batch capping to avoid hogging the thread pool thread:

```csharp
[LogTag("CLEANUP")]
internal sealed class CleanupService : ServiceBase
{
    private readonly int _batchSize = 5_000;
    private readonly ConcurrentDictionary<long, Entry> _entries = new();

    protected override void Setup()
    {
        Interval = TimeSpan.FromHours(1);
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

## Remarks

- `Update()` runs on a **thread pool thread** — do not hold long locks or perform blocking I/O. The thread pool thread is returned as soon as `Update()` returns.
- `Dispose()` uses `Debug.Assert` to catch double-dispose in debug builds.
- Unlike `ManagerBase`, there is no `NotifyWork()`, no stall detection, and no health monitoring. Services are fire-and-forget on their timer.
