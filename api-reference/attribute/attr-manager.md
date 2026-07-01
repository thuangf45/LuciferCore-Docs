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

---

## Notes

- One `[Manager]` per class (`AllowMultiple = false`).
- Not inherited (`Inherited = false`), so child class must declare its own `[Manager]`.
- Manager name is used in logs and manager console commands:
  - `/start managers`
  - `/stop managers`
  - `/restart managers`