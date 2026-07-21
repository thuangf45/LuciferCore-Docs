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

Binds a static property to a config key, sourced from a `.lucifercore` config file (works like a `.env` file).

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
| `key` | Config key, looked up as an environment variable name |
| `defaultValue` | Fallback value used when the key is not found anywhere |

### How mapping works

At startup, `LuciferConfig` runs two steps automatically:

**1. Load `.lucifercore` into environment variables**

LuciferCore looks for a file named `.lucifercore`, in this order:

1. `.lucifercore` (project root)
2. `assets/config/.lucifercore`

The file uses `.env`-style syntax:

```
# comment lines start with #
WWW=assets/client/prod
CERTIFICATE=assets/tools/certificates/server.pfx
CERT_PASSWORD=MyRealPassword123
```

- Blank lines and lines starting with `#` are skipped.
- Each `KEY=VALUE` line is set via `Environment.SetEnvironmentVariable(key, value)` — this **overwrites** any existing process/OS environment variable with the same name.
- If no `.lucifercore` file is found in either location, this step is skipped and existing OS environment variables (if any) are used as-is.

**2. Bind values onto `[Config]` properties**

LuciferCore then scans all `static` properties marked with `[Config]` and, for each one:

1. Reads `Environment.GetEnvironmentVariable(key)` (this reflects whatever `.lucifercore` set in step 1, or a real OS env var if not overwritten).
2. If the env var is not set, falls back to `defaultValue`.
3. If both are `null`, the property is left untouched.
4. Converts the resolved string to the property's type via `Convert.ChangeType` (nullable types are unwrapped automatically).
   - `bool` has special handling: the strings `"1"`, `"true"`, `"TRUE"` are treated as `true` (env vars are always plain strings).
5. Only assigns the property if the converted value actually differs from its current value (avoids unnecessary reflection writes).

If conversion fails for any property (e.g. wrong type, invalid value), the error is caught and logged as `[Config Error] {key}: {message}` — it does not stop other properties from loading.

### Priority summary

```
.lucifercore file value  >  pre-existing OS environment variable  >  DefaultValue
```

(In practice, if the key exists in `.lucifercore`, it always wins, since loading the file overwrites the OS env var of the same name before binding happens.)

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