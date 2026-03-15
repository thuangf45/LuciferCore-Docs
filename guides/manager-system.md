# Manager System

A **Manager** is a background worker that runs on a fixed loop — similar to Unity's `Update()` cycle. Managers are ideal for recurring tasks such as heartbeats, cleanup routines, state broadcasting, and monitoring.

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

## Lifecycle Methods

Extend `ManagerBase` and override these three methods:

### `Setup()`

Called once when the manager starts. Use it to configure timing and initial state.

```csharp
protected override void Setup()
{
    TimeDelay  = 1000; // milliseconds between each Update() call
    ErrorDelay = 1000; // milliseconds to wait after an exception before retrying
}
```

| Property | Description |
|---|---|
| `TimeDelay` | Interval between successive `Update()` calls, in milliseconds |
| `ErrorDelay` | Cooldown period after an unhandled exception, in milliseconds |

### `Update()`

Called repeatedly on the configured interval. This is where your recurring logic lives.

```csharp
protected override void Update()
{
    Lucifer.Log(this, "Master is running....");
}
```

### `Dispose(bool disposing)`

Called when the manager is stopped or the host shuts down. Always call `base.Dispose(disposing)`.

```csharp
protected override void Dispose(bool disposing)
{
    base.Dispose(disposing);
}
```

---

## Full Example

```csharp
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase
{
    protected override void Setup()
    {
        TimeDelay  = 1000;
        ErrorDelay = 1000;
    }

    protected override void Update()
    {
        Lucifer.Log(this, "Master is running....");
    }

    protected override void Dispose(bool disposing)
    {
        base.Dispose(disposing);
    }
}
```

---

## Managing from the Console

Once running, managers can be controlled via built-in console commands:

```
/start managers
/stop managers
/restart managers
```

See [Entry Point](../getting-started/entry-point.md) for the full command reference.
