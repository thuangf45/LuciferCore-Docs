# Manager System

A **Manager** is a background worker.

It runs in a workload-based loop:
- sleeps when no work
- wakes when new work arrives

Use it for:
- heartbeat tasks
- cleanup jobs
- queue processing
- monitoring
- state broadcast

---

## `[Manager]` attribute

```csharp
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase { ... }
```

| Parameter | Type | Meaning |
|---|---|---|
| `name` | `string` | Manager name |

LuciferCore auto-discovers managers and starts them when the host runs.

---

## How loop works

Manager loop is **workload-driven** (not fixed timer).

- If `workload <= 0`: manager waits (sleep state)
- If `workload > 0`: manager runs `Update()`
- `NotifyWork()` wakes manager immediately when new work comes

`Stop()` pauses processing.  
`Start()` resumes quickly (no heavy respawn flow).

---

## Main lifecycle methods

### `Setup()`
Runs once when manager thread starts.

```csharp
protected override void Setup()
{
    // one-time init
}
```

### `Update()`
Main processing method.

```csharp
protected override void Update()
{
    Lucifer.Log(this, "Manager is running...");
}
```

### `workload`
A protected `volatile int` value for pending work count.

Example:

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
Call when new work is added.

```csharp
public void Enqueue(Job job)
{
    _queue.Enqueue(job);
    Interlocked.Increment(ref workload);
    NotifyWork();
}
```

### `Cleanup()` and `Reload()`
Used in restart flow:

1. `Cleanup()` release old resources
2. `Reload()` init again

```csharp
protected override void Cleanup()
{
    // release resources
}

protected override void Reload()
{
    // init again
}
```

### `OnStarted()` and `OnStopped()`
Hooks for logging/notifications.

```csharp
protected override void OnStarted() => Lucifer.Log(this, "Manager started");
protected override void OnStopped() => Lucifer.Log(this, "Manager stopped");
```

---

## Full example

```csharp
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase
{
    protected override void Setup()
    {
        Lucifer.Log(this, "Setup complete");
    }

    protected override void Update()
    {
        Lucifer.Log(this, "Manager is running...");
    }

    protected override void Cleanup()
    {
        // release resources
    }

    protected override void Reload()
    {
        // reinitialize state
    }
}
```

---

## Console commands

```text
/start managers
/stop managers
/restart managers
```