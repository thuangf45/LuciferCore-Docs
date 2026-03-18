# [Service]

**Namespace:** `LuciferCore.Attributes`

Registers a class as a service. LuciferCore auto-discovers all `[Service]`-decorated classes and starts them as part of the service system when `Lucifer.Run()` is called — identical to how `[Manager]` works for managers.

The decorated class must extend `ServiceBase`.

---

## Declaration

```csharp
[AttributeUsage(AttributeTargets.Class, Inherited = false, AllowMultiple = false)]
public class ServiceAttribute : Attribute
```

---

## Constructor

```csharp
public ServiceAttribute(string name)
```

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Human-readable identifier for this service instance |

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Name` | `ByteString` | UTF-8 encoded service name |

---

## Usage

```csharp
[Service("CleanupService")]
internal sealed class CleanupService : ServiceBase
{
    protected override void Setup()
    {
        Interval = TimeSpan.FromHours(1);
    }

    protected override void Update()
    {
        // periodic cleanup logic
        Workload = _entries.Count;
    }
}
```

`Lucifer.Run()` auto-discovers the class, instantiates it, and calls `Start()` — no manual registration needed.

---

## Remarks

- Only one `[Service]` attribute is allowed per class (`AllowMultiple = false`).
- The attribute is not inherited (`Inherited = false`) — subclasses must declare their own attribute.
- The service name is used in console commands (`/start services`, `/stop services`, `/restart services`) and log output.
