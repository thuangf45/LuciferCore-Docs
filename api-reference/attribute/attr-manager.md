# [Manager]

**Namespace:** `LuciferCore.Attributes`

Registers a class as a background manager. LuciferCore auto-discovers all `[Manager]`-decorated classes and starts them as part of the async loop system when `Lucifer.Run()` is called.

The decorated class must extend `ManagerBase`.

---

## Declaration

```csharp
[AttributeUsage(AttributeTargets.Class, Inherited = false, AllowMultiple = false)]
public class ManagerAttribute : Attribute
```

---

## Constructor

```csharp
public ManagerAttribute(string name)
```

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Human-readable identifier for this manager instance |

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Name` | `ByteString` | UTF-8 encoded manager name |

---

## Usage

```csharp
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase
{
    protected override void Setup()
    {
        TimeDelay  = 1000; // ms between each Update()
        ErrorDelay = 1000; // ms cooldown after an exception
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

## Lifecycle

| Method | When called |
|---|---|
| `Setup()` | Once, immediately when the manager starts |
| `Update()` | Repeatedly, every `TimeDelay` milliseconds |
| `Dispose(bool)` | Once, when the manager is stopped or the host shuts down |

---

## Remarks

- Only one `[Manager]` attribute is allowed per class (`AllowMultiple = false`).
- The attribute is not inherited (`Inherited = false`) — subclasses must declare their own attribute.
- The manager name is used in console commands (`/start managers`, `/stop managers`, `/restart managers`) and log output.
- If `Update()` throws an unhandled exception, execution pauses for `ErrorDelay` milliseconds before resuming. Use `[Safe]` on individual methods within a manager if finer-grained error handling is needed.
