# ManagerBase

**Namespace:** `LuciferCore.Manager`

Abstract base class for all LuciferCore managers. Implements an immortal background thread that runs for the lifetime of the process — `Stop()`/`Start()` pause and resume processing without killing the thread. The thread is spawned once on first `Start()` and terminates only when `Dispose()` is called.

```csharp
public abstract class ManagerBase : IDisposable
```

---

## Thread Model

```
Start()
    → thread spawned (once)
    → Setup()           — one-time initialization on thread start
    → loop:
        IsRunning?
            true  → Update() → Sleep(TimeDelay)
            false → WaitForWakeSignal()
    → Cleanup()         — called when thread exits
Dispose()
    → signals thread to exit
    → Dispose(bool)     — your cleanup hook
```

`Stop()` pauses the loop without killing the thread — `Start()` resumes it instantly. `Restart()` runs `Cleanup()` → `Reload()` → `Start()` for hot-reload scenarios without re-spawning the thread.

---

## Lifecycle API

```csharp
void Start()
void Stop()
void Restart()
void Dispose()
```

| Method | Description |
|---|---|
| `Start()` | Spawns the background thread on first call. Subsequent calls resume if paused. No-op if already running or disposed. |
| `Stop()` | Pauses the update loop. Thread stays alive. Calls `OnStopping()` → `OnStopped()`. |
| `Restart()` | Stops → `Cleanup()` → `Reload()` → `Start()`. Useful for hot-reloading state or config. |
| `Dispose()` | Signals thread exit, removes budget from `FinanceManager`, runs `Dispose(bool)` then `OnDisposed()`. |

---

## State & Workload

| Member | Type | Description |
|---|---|---|
| `Workload` | `int` (volatile) | Current load metric — e.g., queue depth, session count. Updated by the derived class in `Update()`. Read by `FinanceManager` to compute adaptive delay. |
| `IsRunning` | `bool` | `true` while not paused. |
| `IsStalled` | `bool` (volatile) | Set by the derived class when it cannot make progress (e.g., no worker slots available, channel at capacity). Automatically cleared at the start of each `Update()` cycle. |

---

## Delay & Scheduling

| Property | Default | Description |
|---|---|---|
| `TimeDelay` | `500` ms | Sleep duration between `Update()` calls. Recalculated each cycle by `FinanceManager` via `Lucifer.GetBudgetedDelay()`. |
| `ErrorDelay` | `1000` ms | Sleep duration after an unhandled exception in `Update()`. |
| `DelayDefault` | `250` ms | Fallback delay when no budget is registered with `FinanceManager`. |
| `MinimumSleepMs` | `1` ms | Hard floor on computed sleep — prevents busy-spin. |
| `ThreadPriority` | `Lowest` | OS thread priority. Set this in the derived constructor before `Start()` is called. |

Adaptive delay is driven by `FinanceManager`. Register a budget inside `Setup()`:

```csharp
protected override void Setup()
{
    // minDelay, maxDelay, workloadThreshold, smoothingFactor
    Lucifer.AddBudget(this, 50, 1000, 5000, 0.2f);
}
```

When `Workload` stays at zero long enough, the delay becomes `Timeout.Infinite` — the thread parks completely and wakes only via `NotifyWork()`.

---

## Wake Notification

```csharp
public void NotifyWork()
```

Wakes the manager from sleep immediately. Thread-safe — safe to call from any thread, including network callbacks. Best practice: call whenever new work is enqueued, throttled by count to avoid excessive wakes:

```csharp
var count = _queue.Count;
if (count <= 2 || (count & 15) == 0) NotifyWork();
```

---

## Health Monitoring

```csharp
public ManagerStats GetHealthStats()  // full metrics snapshot
public string GetHealthSummary()      // compact one-line string, zero allocations
```

Health is recalculated every **10 seconds** automatically:

| Condition | Status |
|---|---|
| `ErrorsPerSecond > 10` | `Unhealthy` |
| `StallsPerSecond > 5` | `Degraded` |
| Not running | `Paused` |
| `Workload == 0` | `Idle` |
| Otherwise | `Healthy` |

---

## Override Points

| Method | Required | Description |
|---|---|---|
| `Update()` | **Yes** | Main work body. Called in a tight loop while running. |
| `Setup()` | No | One-time init on thread start. Register `AddBudget` here. |
| `Cleanup()` | No | Called on `Restart()` and when the thread exits. Release resources here. |
| `Reload()` | No | Called during `Restart()` after `Cleanup()`. Re-initialize state or config. |
| `Dispose(bool)` | No | Override for custom disposal. Called from `Dispose()`. |
| `ShouldContinueRunning()` | No | Return `true` to keep processing after `Stop()` — useful for draining a channel before fully pausing. |
| `OnStarting()` / `OnStarted()` | No | Called before / after the manager resumes. |
| `OnStopping()` / `OnStopped()` | No | Called before / after the manager pauses. |
| `OnRestarting()` / `OnRestarted()` | No | Called before / after a restart cycle. |
| `OnDisposing()` / `OnDisposed()` | No | Called at the start / end of `Dispose()`. |

---

## ManagerStats

Snapshot of all runtime metrics. Obtained via `GetHealthStats()`. Safe to read from any thread.

| Property | Type | Description |
|---|---|---|
| `HealthStatus` | `HealthStatus` | Current health classification |
| `CurrentWorkload` | `int` | Latest workload reading |
| `PeakWorkload` | `long` | Highest workload observed since first start |
| `TotalUpdateCount` | `long` | Total successful `Update()` invocations |
| `TotalErrorCount` | `long` | Total exceptions caught in `Update()` |
| `TotalStallCount` | `long` | Total stall events recorded |
| `UpdatesPerSecond` | `double` | Rolling throughput rate since start |
| `ErrorsPerSecond` | `double` | Rolling error rate since start |
| `StallsPerSecond` | `double` | Rolling stall rate since start |
| `CpuUsagePercent` | `double` | Active time ÷ (active + sleep + idle) × 100 |
| `Uptime` | `TimeSpan` | Elapsed time since first `Start()` |
| `IsHealthy` | `bool` | `true` when status is `Healthy` or `Idle` |
| `IdleCount` | `long` | Times the thread blocked waiting for `NotifyWork()` |
| `WakeNotificationCount` | `long` | Total wake signals sent via `NotifyWork()` |
| `StartCount` / `StopCount` / `RestartCount` | `long` | Lifecycle event counters |
| `StartTime` / `LastStartTime` / `LastStopTime` | `DateTime` | Lifecycle timestamps |

---

## HealthStatus

```csharp
public enum HealthStatus
{
    NotStarted,  // Thread not yet spawned
    Healthy,     // Actively processing work
    Idle,        // Running but no workload
    Paused,      // Stopped but not disposed
    Degraded,    // High stall rate (> 5/sec)
    Unhealthy,   // High error rate (> 10/sec)
    Disposed     // Dispose() has been called
}
```

---

## ManagerType

Used internally for categorization and routing via `Lucifer.NotifyWork(ManagerType)`.

```csharp
public enum ManagerType
{
    RateLimiter,
    Session,
    Log,
    Database,
    Simulation,
    Dispatch,
    Finance
}
```
