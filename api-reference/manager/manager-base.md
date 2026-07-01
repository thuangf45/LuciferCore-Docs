# ManagerBase

**Namespace:** `LuciferCore.Manager`

Abstract base class for long-running background managers. Spawns a single dedicated thread (via a shared high-performance worker pool) on first `Start()`. `Stop()`/`Start()` pause and resume the loop without re-spawning the thread.

```csharp
public abstract class ManagerBase
```

---

## Thread Model

```csharp
Start()
    → thread spawned once (s_highPerformanceWorker)
    → Setup()                — one-time init on thread start
    → loop:
        workload <= 0 → wait for NotifyWork()
        Update()
        workload > 0  → wait up to 1ms, then loop again
```

`Stop()` pauses processing (`IsRunning` becomes `false`) but the loop and thread keep running internally — the derived class is responsible for checking `IsRunning` inside `Update()` if it needs to skip work while stopped. `Restart()` calls `Cleanup()` → `Reload()` → re-`Start()` (only if it was running).

---

## Properties

| Member | Type | Description |
|---|---|---|
| `ThreadPriority` | `ThreadPriority` (protected, default `Lowest`) | OS thread priority. Set in the derived constructor before `Start()` is called. |
| `workload` | `int` (protected, `volatile`) | Current load metric (e.g. queue size, active sessions). Set by the derived class inside `Update()`. When `<= 0`, the thread parks until `NotifyWork()` is called. |
| `IsRunning` | `bool` (protected) | `true` while not paused. |

---

## Lifecycle API

```csharp
public void Start()
public void Stop()
public void Restart()
public void NotifyWork()
```

| Method | Description |
|---|---|
| `Start()` | Spawns the background thread on first call (no-op on later calls). Sets running state, calls `OnStarted()`. |
| `Stop()` | Clears running state, calls `OnStopped()`. Thread keeps looping but parked/idling depending on `workload`. |
| `Restart()` | If running: `Stop()` → `Cleanup()` → `Reload()` → `Start()`. If not running: just `Cleanup()` → `Reload()`. |
| `NotifyWork()` | Wakes the manager from its wait. Thread-safe — safe to call from any thread. |

---

## Override Points

| Method | Required | Description |
|---|---|---|
| `Update()` | **Yes** | Main work body, called repeatedly while not waiting. |
| `Setup()` | No | One-time initialization, called once when the thread starts. |
| `Cleanup()` | No | Called during `Restart()`, before `Reload()`. |
| `Reload()` | No | Called during `Restart()`, after `Cleanup()`. |
| `OnStarted()` | No | Called from `Start()` after the running state is set. Default logs `"Started"`. |
| `OnStopped()` | No | Called from `Stop()` after the running state is cleared. Default logs `"Stopped"`. |

---

## Notes

- There is no `IDisposable` implementation, no health monitoring/stats API, no adaptive delay/`FinanceManager` integration, and no `ManagerStats`/`HealthStatus`/`ManagerType` types in the current version — all of that has been removed.
- Exceptions thrown from `Setup()` abort the thread entirely (loop never starts). Exceptions thrown from `Update()` are caught, logged, and the loop continues.
- `workload` is read/written directly by derived classes (no public exposure) — there's no `Workload` property and no peak/throughput tracking.

---