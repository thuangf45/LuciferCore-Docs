# ManagerBase

**Namespace:** `LuciferCore.Manager`

`ManagerBase` is the base class for long-running background workers.

A manager starts one worker thread on first `Start()`.  
Later `Stop()` / `Start()` pause and resume without creating a new thread.

```csharp
public abstract class ManagerBase
```

---

## Thread model (simple)

```text
Start()
 -> create worker thread (first time only)
 -> call Setup() once
 -> run loop:
    if workload <= 0: wait
    call Update()
    if workload > 0: short wait (~1ms), continue
```

`Stop()` changes running state to paused.  
`Start()` resumes it.

`Restart()` flow:
- if running: `Stop() -> Cleanup() -> Reload() -> Start()`
- if not running: `Cleanup() -> Reload()`

---

## Properties

| Member | Type | Meaning |
|---|---|---|
| `ThreadPriority` | `ThreadPriority` (protected) | Worker thread priority |
| `workload` | `int` (protected, `volatile`) | Current pending work amount |
| `IsRunning` | `bool` (protected) | Running/paused state |

---

## Lifecycle methods

```csharp
public void Start()
public void Stop()
public void Restart()
public void NotifyWork()
```

| Method | Meaning |
|---|---|
| `Start()` | Start manager (first call creates thread) |
| `Stop()` | Pause manager |
| `Restart()` | Cleanup and reload manager |
| `NotifyWork()` | Wake manager when new work arrives |

`NotifyWork()` is thread-safe.

---

## Override points

| Method | Required | Meaning |
|---|---|---|
| `Update()` | Yes | Main loop logic |
| `Setup()` | No | One-time init on thread start |
| `Cleanup()` | No | Cleanup before reload |
| `Reload()` | No | Re-init after cleanup |
| `OnStarted()` | No | Hook after start |
| `OnStopped()` | No | Hook after stop |

---

## Notes

- Keep `Update()` safe and lightweight.
- If `Update()` throws, error is logged and loop continues.
- `workload` is managed by derived class logic.
- This API is intentionally minimal (no extra stats/health model in current version).