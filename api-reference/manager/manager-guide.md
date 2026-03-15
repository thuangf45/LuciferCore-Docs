# Writing a Custom Manager

This guide walks through implementing a manager by inheriting `ManagerBase`. The built-in system managers (`LogManager`, `SessionManager`, `RateLimiterManager`, `SimulationManager`, `DispatchManager`, `FinanceManager`) are used throughout as real-world reference examples.

---

## Registering with the System

Decorate the class with `[Manager]` to register it with LuciferCore. The `name` string is the unique identifier used to look up the manager at runtime.

```csharp
using LuciferCore.Attributes;

[Manager("MyManager")]
internal sealed class MyManager : ManagerBase
{
    // ...
}
```

**Namespace:** `LuciferCore.Attributes`

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Unique identifier for this manager. Used internally as a `ByteString` key for fast lookup. |

The attribute is `AllowMultiple = false` — one name per class. Without it the manager will not be discovered or registered by the system.

---

## Minimal Implementation

Every manager needs the `[Manager]` attribute and exactly one override — `Update()`:

```csharp
[Manager("MyManager")]
internal sealed class MyManager : ManagerBase
{
    public MyManager() => ThreadPriority = ThreadPriority.BelowNormal;

    protected override void Setup()
    {
        ErrorDelay = 1000;
        Lucifer.AddBudget(this, minDelay: 100, maxDelay: 5000, threshold: 1000, smoothingFactor: 0.2f);
    }

    protected override void Update()
    {
        // your work here
        Workload = /* current queue depth, count, etc. */;
    }
}
```

Then register and start it through `Lucifer`:

```csharp
Lucifer.RegisterManager<MyManager>();
Lucifer.Start<MyManager>();
```

---

## Setting Thread Priority

Set `ThreadPriority` in the constructor — before the thread spawns. All built-in managers follow this pattern:

```csharp
// LogManager — lowest priority, background I/O
public LogManager() => ThreadPriority = ThreadPriority.Lowest;

// SimulationManager — slightly higher, drives the event loop
public SimulationManager() => ThreadPriority = ThreadPriority.BelowNormal;

// DispatchManager — same, processes network jobs
public DispatchManager() => ThreadPriority = ThreadPriority.BelowNormal;
```

Choose based on how latency-sensitive the manager is relative to other threads in the process.

---

## Registering an Adaptive Budget

Call `Lucifer.AddBudget()` inside `Setup()`. The `FinanceManager` reads `Workload` each cycle and adjusts the sleep delay automatically — more work means shorter sleep, idle means eventually parking the thread indefinitely.

```csharp
protected override void Setup()
{
    ErrorDelay = 1000;
    Lucifer.AddBudget(this,
        minDelay:        50,      // fastest possible sleep (ms) under full load
        maxDelay:        5000,    // slowest sleep (ms) when nearly idle
        threshold:       10_000,  // workload level considered "fully loaded"
        smoothingFactor: 0.2f     // EMA factor — lower = smoother, slower to react
    );
}
```

**Real examples:**

```csharp
// LogManager — high throughput bursts, moderate idle
Lucifer.AddBudget(this, 250, 1000, 50_000, 0.2f);

// SessionManager — slow-changing, long idle periods acceptable
Lucifer.AddBudget(this, 300_000, 600_000, 50_000, 0.1f);

// RateLimiterManager — infrequent cleanup, very long idle ok
Lucifer.AddBudget(this, 5000, 60_000, 50_000, 0.1f);

// SimulationManager / DispatchManager — latency sensitive, near-zero min delay
Lucifer.AddBudget(this, 0, 200, alertThreshold, 0.2f);
```

---

## Reporting Workload

Set `Workload` at the end of each `Update()` to reflect the current amount of pending work. `FinanceManager` reads this value to compute the next sleep delay:

```csharp
// SessionManager — count of active sessions
protected override void Update()
{
    _ = Cleanup(_batchSize);
    Workload = _sessions.Count;
}

// RateLimiterManager — count of tracked rate states
protected override void Update()
{
    Cleanup(_batchSize);
    Workload = _sessionStates.Count;
}
```

For channel-based managers, decrement atomically after processing to keep the value accurate under concurrent writes:

```csharp
// DispatchManager
protected override void Update()
{
    var processed = Tick(KPI);
    if (processed > 0)
    {
        var remaining = Interlocked.Add(ref workload, -processed);
        if (remaining < 0) Interlocked.Exchange(ref workload, 0);
    }
}
```

---

## Signaling Stalls

Set `IsStalled = true` when the manager has pending work but cannot progress — for example, when the worker pool is saturated. `ManagerBase` clears this flag automatically at the start of each `Update()`.

```csharp
// SimulationManager — stalls when events are queued but no worker slots available
protected override void Update()
{
    MergeIncoming();
    KPI = Math.Clamp(MaxWorkload, 0, _batchSize);
    Workload = Tick(KPI);

    if (Workload > 0 && KPI == 0)
        IsStalled = true;
}

// DispatchManager — same pattern, stalls when channel has jobs but KPI is zero
protected override void Update()
{
    var processed = Tick(KPI);
    // ...
    if (Workload > 0 && KPI == 0) IsStalled = true;
}
```

`FinanceManager` detects stalled managers and resets their delay to `DelayDefault` to allow faster recovery.

---

## Waking the Manager from External Threads

When work arrives from outside the manager's own thread (e.g., a network callback, a producer), call `NotifyWork()` to wake it from sleep immediately. Throttle the call to avoid signaling on every single item:

```csharp
// LogManager — called from any thread writing a log entry
public void Log<T>(object instance, ReadOnlySpan<T> message, LogLevel level) where T : unmanaged
{
    if (_writer.TryWrite(new(instance, level, msg)))
    {
        workload++;
        if (workload <= 2 || (workload & 31) == 0) NotifyWork();
    }
}

// SessionManager — called when a new session is created
var count = _sessions.Count;
if (count <= 2 || (count & 15) == 0) NotifyWork();
```

The `(count & N) == 0` pattern limits wake signals to every N items, avoiding unnecessary context switches under high-throughput load.

---

## Processing in Batches

Always cap work per `Update()` cycle with a `_batchSize` field. This keeps the update loop responsive to sleep/wake signals and prevents a single manager from monopolizing the thread:

```csharp
private readonly int _batchSize = 5_000;

protected override void Update()
{
    var removed = Cleanup(_batchSize);
    Workload = _sessions.Count;
}

private int Cleanup(int batchSize)
{
    var removed = 0;
    foreach (var kv in _sessions)
    {
        if (removed >= batchSize) break;
        // ... remove expired entries
        removed++;
    }
    return removed;
}
```

---

## Draining Before Pause

Override `OnStopped()` to flush remaining work before the manager fully pauses. This prevents data loss when `Stop()` is called mid-stream:

```csharp
// LogManager — drains all pending log entries before pausing
protected override void OnStopped()
{
    var drained = DrainLogs(100_000);
    if (drained > 0) _logStream?.Flush(true);
    workload = 0;
}
```

---

## Staying Alive After Stop

Override `ShouldContinueRunning()` to keep the update loop running even after `Stop()` — for example, to drain a channel before parking:

```csharp
// LogManager — keeps running until the channel signals completion
protected override bool ShouldContinueRunning() => !_reader.Completion.IsCompleted;

// DispatchManager — keeps running while jobs remain
protected override bool ShouldContinueRunning() => _reader.Count > 0 || workload > 0;
```

Without this, the thread blocks on `WaitForWakeSignal()` immediately after `Stop()`, which may leave items in the queue unprocessed.

---

## Cleanup on Dispose

Override `Dispose(bool)` to release unmanaged resources. Always call `base.Dispose(disposing)` at the end:

```csharp
// LogManager — completes the channel and closes the file stream
protected override void Dispose(bool disposing)
{
    _writer.TryComplete();
    CloseWriters();
    base.Dispose(disposing);
}

// DispatchManager — completes the channel and returns pooled packets
protected override void Dispose(bool disposing)
{
    _writer.Complete();
    while (_reader.TryRead(out var job))
    {
        if (job.Data is PacketModel packet) Lucifer.Return(packet);
    }
    base.Dispose(disposing);
}
```

---

## Complete Example

Putting it all together — a queue-draining manager with adaptive sleep and cooperative batching:

```csharp
[Manager("MyManager")]
internal sealed class MyManager : ManagerBase
{
    private readonly int _batchSize = 1_000;
    private readonly ConcurrentQueue<WorkItem> _queue = new();

    public MyManager() => ThreadPriority = ThreadPriority.BelowNormal;

    protected override void Setup()
    {
        ErrorDelay = 1000;
        Lucifer.AddBudget(this, minDelay: 0, maxDelay: 500, threshold: 5_000, smoothingFactor: 0.2f);
    }

    protected override void Update()
    {
        var processed = 0;
        while (processed < _batchSize && _queue.TryDequeue(out var item))
        {
            Process(item);
            processed++;
        }

        var remaining = Interlocked.Add(ref workload, -processed);
        if (remaining < 0) Interlocked.Exchange(ref workload, 0);

        if (Workload > 0 && processed == 0) IsStalled = true;
    }

    public void Enqueue(WorkItem item)
    {
        _queue.Enqueue(item);
        var count = Interlocked.Increment(ref workload);
        if (count <= 2 || (count & 15) == 0) NotifyWork();
    }

    private void Process(WorkItem item) { /* ... */ }
}
```
