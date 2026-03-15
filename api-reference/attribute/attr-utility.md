# Utility Attributes

**Namespace:** `LuciferCore.Attributes`

Utility attributes handle cross-cutting concerns that are not part of the core request/response pipeline: interactive console commands and structured log tagging.

---

## `[ConsoleCommand]`

Registers a static method as an interactive console command. LuciferCore auto-discovers all `[ConsoleCommand]`-decorated methods at startup and makes them available in the console host launched by `Lucifer.Run()`.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class ConsoleCommandAttribute : Attribute
```

### Constructor

```csharp
public ConsoleCommandAttribute(string name, string description = "")
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `name` | `string` | — | The exact command string the user types in the console (e.g. `"/start proxy"`) |
| `description` | `string` | `""` | Short description shown in the help listing |

### Properties

| Property | Type | Description |
|---|---|---|
| `Name` | `ByteString` | UTF-8 encoded command string |
| `Description` | `ByteString` | UTF-8 encoded description |

### Usage

```csharp
[ConsoleCommand("/start proxy", "Start proxy")]
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private static void CmdStartProxy() => Start();

[ConsoleCommand("/stop proxy", "Stop proxy")]
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private static void CmdStopProxy() => Stop();
```

### Remarks

- The decorated method must be `static` and take no parameters.
- Command names are case-sensitive. Use a consistent convention such as `/verb noun`.
- Custom commands are discovered from any class in your assembly — no manual registration needed.
- LuciferCore also ships with built-in commands for host, servers, and managers. See [Entry Point](../getting-started/entry-point.md) for the full built-in command reference.

---

## `[LogTag]`

Assigns a custom log tag identifier to a class. When `Lucifer.Log()` is called with an instance of the decorated class, the tag is used as the log prefix instead of the default class name.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Class, Inherited = false)]
public class LogTagAttribute : Attribute
```

### Constructor

```csharp
public LogTagAttribute(string tag)
```

| Parameter | Type | Description |
|---|---|---|
| `tag` | `string` | The log tag string used as a prefix in log output |

### Properties

| Property | Type | Description |
|---|---|---|
| `Tag` | `ByteString` | UTF-8 encoded log tag |

### Usage

```csharp
[LogTag("MASTER")]
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase
{
    protected override void Update()
    {
        Lucifer.Log(this, "Tick"); // output: [MASTER] Tick
    }
}
```

### Remarks

- `[LogTag]` only applies to classes (`AttributeTargets.Class`).
- The attribute is not inherited (`Inherited = false`) — each class must declare its own tag if needed.
- The tag is encoded as a `ByteString` once at startup and reused on every log call without allocation.
- If `[LogTag]` is not present, `Lucifer.Log()` uses a default identifier derived from the class.
