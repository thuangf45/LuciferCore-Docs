# Manager System

A **Manager** is a background worker that runs on a continuous, workload-driven loop. Unlike fixed-interval tickers, a Manager sleeps when there's nothing to do and wakes up instantly when new work arrives — ideal for heartbeats, cleanup routines, state broadcasting, queue processing, and monitoring tasks.

---

## The `[Manager]` Attribute

```csharp
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase { ... }
```

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | A human-readable identifier for this manager |

LuciferCore auto-discovers and starts all `[Manager]`-decorated classes when `Lucifer.Run()` is called.

---

## How the Loop Works

Each Manager spawns its worker thread exactly once (on first `Start()`) and keeps it alive for the lifetime of the application. The loop logic is workload-driven, not timer-driven:

- If `workload <= 0`, the thread blocks on an internal wait handle until `NotifyWork()` is called.
- If `workload > 0`, `Update()` is called immediately, then the thread does a short, non-blocking yield (`1ms`) before checking again.
- `Stop()` pauses processing without killing the thread; `Start()` resumes it instantly — no respawn cost.

This means there's no fixed polling interval to configure: the Manager reacts to actual workload instead of running on a blind timer.

---

## Lifecycle Methods

Extend `ManagerBase` and override the methods relevant to your use case:

### `Setup()`
Called once when the manager's thread starts, before the loop begins. Use it for one-time initialization.

```csharp
protected override void Setup()
{
    // Initialize state, open connections, etc.
}
```

### `Update()`
The core loop method — called repeatedly whenever `workload > 0` (or once per wake if you don't track workload). This is where your recurring logic lives.

```csharp
protected override void Update()
{
    Lucifer.Log(this, "Master is running....");
}
```

### `workload`

A protected `volatile int` field representing the current amount of pending work (e.g. queue size, active sessions). Update it inside `Update()` to reflect real load — the loop uses this value to decide whether to keep processing or go back to sleep.

```csharp
protected override void Update()
{
    while (_queue.TryDequeue(out var item))
    {
        Process(item);
        Interlocked.Decrement(ref workload);
    }
}
```

### `NotifyWork()`

Call this whenever new work is enqueued, to wake the manager's thread immediately instead of waiting for its next poll.

```csharp
public void Enqueue(Job job)
{
    _queue.Enqueue(job);
    Interlocked.Increment(ref workload);
    NotifyWork();
}
```

### `Cleanup()` / `Reload()`

Called during `Restart()`: `Cleanup()` runs first to release resources, then `Reload()` runs to reinitialize state, before the manager starts again.

```csharp
protected override void Cleanup()
{
    // Release resources before restart
}

protected override void Reload()
{
    // Reinitialize state after cleanup
}
```

### `OnStarted()` / `OnStopped()`

Hooks called right after the manager starts or stops, useful for logging or notifying other systems.

```csharp
protected override void OnStarted() => Lucifer.Log(this, "Master started");
protected override void OnStopped() => Lucifer.Log(this, "Master stopped");
```

---

## Full Example

```csharp
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase
{
    protected override void Setup()
    {
        Lucifer.Log(this, "Master setup complete");
    }

    protected override void Update()
    {
        Lucifer.Log(this, "Master is running....");
    }

    protected override void Cleanup()
    {
        // Release resources
    }

    protected override void Reload()
    {
        // Reinitialize state
    }
}
```

---

## Managing from the Console

Once running, managers can be controlled via built-in console commands:

```csharp
/start managers
/stop managers
/restart managers
```

---