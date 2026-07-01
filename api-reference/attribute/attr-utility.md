# Core Attributes

**Namespace:** `LuciferCore.Attributes`

Core attributes are optional helpers.

They help with:
- plugin discovery
- console commands
- config binding
- log naming
- layout field order

The system can run without them.

---

## `[Plugin]`

Marks a class as a plugin with metadata.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
public class PluginAttribute : Attribute { }
```

### Usage

```csharp
[Plugin("Renderer", "MyCustomRenderer")]
public class MyCustomRenderer : IRenderer
{
    // ...
}
```

| Field | Meaning |
|---|---|
| `pluginType` | Plugin group (example: `"Renderer"`, `"AI"`, `"Audio"`) |
| `name` | Plugin display name |

---

## `[ConsoleCommand]`

Registers a method as a console command.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Method, AllowMultiple = true)]
public class ConsoleCommandAttribute : Attribute { }
```

### Usage

```csharp
[ConsoleCommand("/start proxy", "Start proxy")]
private static void CmdStartProxy() => Start();

[ConsoleCommand("/stop proxy", "Stop proxy")]
private static void CmdStopProxy() => Stop();

// same method, multiple aliases
[ConsoleCommand("/stop")]
[ConsoleCommand("/shutdown", "Shut down the server")]
private static void CmdStop() => Stop();
```

| Field | Meaning |
|---|---|
| `name` | Command text user types |
| `description` | Help text (optional) |

Notes:
- Method should be `static` with no parameters.
- `AllowMultiple = true` means one method can have many aliases.

---

## `[Config]`

Binds a static property to a config key.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class ConfigAttribute : Attribute { }
```

### Usage

```csharp
[Config("WWW", "assets/client/dev")]
private static string s_staticContentPath { get; set; } = string.Empty;

[Config("CERTIFICATE", "assets/tools/certificates/server.pfx")]
private static string s_certPath { get; set; } = string.Empty;

[Config("CERT_PASSWORD", "RootCA!SecureKey@Example2025Strong")]
private static string s_certPassword { get; set; } = string.Empty;
```

| Field | Meaning |
|---|---|
| `key` | Config key |
| `defaultValue` | Fallback value when key is missing |

---

## `[LogTag]`

Sets a custom log tag for a class.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Class, Inherited = false)]
public class LogTagAttribute : Attribute { }
```

### Usage

```csharp
[LogTag("MASTER")]
[Manager("MasterManager")]
public class ManagerMaster : ManagerBase
{
    protected override void Update()
    {
        Lucifer.Log(this, "Tick"); // [MASTER] Tick
    }
}
```

If not set, log uses default class-based name.

---

## `[LayoutIndex]`

Sets property order in fixed layout models (for serialization/deserialization).

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Property, AllowMultiple = false)]
public class LayoutIndexAttribute : Attribute { }
```

### Usage

```csharp
public class PlayerLayout
{
    [LayoutIndex(0)]
    public int Id { get; set; }

    [LayoutIndex(1)]
    public float PosX { get; set; }

    [LayoutIndex(2)]
    public float PosY { get; set; }
}
```

`index` is the field order.

---

## Notes

- All attributes in this page are optional helpers.
- `[ConsoleCommand]` is the only one here that supports multiple attributes on one target.
- `[LogTag]` is not inherited by child classes.
- Some text fields are stored internally as `ByteString` for efficient reuse.