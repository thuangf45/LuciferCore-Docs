# [Manager]

**Namespace:** `LuciferCore.Attributes`

Marks a class as a background manager.

LuciferCore auto-discovers all `[Manager]` classes and runs them in the manager loop system.

The class must inherit `ManagerBase`.

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

| Parameter | Type | Meaning |
|---|---|---|
| `name` | `string` | Manager name |

---

## Properties

| Property | Type | Meaning |
|---|---|---|
| `Name` | `ByteString` | UTF-8 manager name |
| `Order` | `int` | Execution/initialization order. Managers run in ascending `Order` (lowest first). Default is `0`. Managers with the same `Order` fall back to discovery order. |

---

## Usage

```csharp
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase
{
    protected override void Setup()
    {
        // one-time setup
    }

    protected override void Update()
    {
        Lucifer.Log(this, "Master is running...");
    }
}
```

### Controlling run order

Use `Order` when a manager must run before or after another one:

```csharp
[Manager("NetworkManager", Order = 0)]
public class ManagerNetwork : ManagerBase
{
    // Runs first
}

[Manager("GameplayManager", Order = 10)]
public class ManagerGameplay : ManagerBase
{
    // Runs after NetworkManager
}
```

---

## Notes

- One `[Manager]` per class (`AllowMultiple = false`).
- Not inherited (`Inherited = false`), so child class must declare its own `[Manager]`.
- Manager name is used in logs and manager console commands:
  - `/start managers`
  - `/stop managers`
  - `/restart managers`
- `Order` determines run/init sequence across all discovered managers (ascending, lowest runs first); leave unset (`0`) if order doesn't matter.