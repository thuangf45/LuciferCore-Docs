# [Config]

**Namespace:** `LuciferCore.Attributes`

Binds a static property to a configuration key. LuciferCore reads all `[Config]`-decorated properties at startup and injects the resolved value before any server or manager is initialized.

---

## Declaration

```csharp
[AttributeUsage(AttributeTargets.Property)]
public class ConfigAttribute : Attribute
```

---

## Constructor

```csharp
public ConfigAttribute(string key, string? defaultValue = null)
```

| Parameter | Type | Description |
|---|---|---|
| `key` | `string` | The configuration key to look up (e.g. from environment variables or a config source) |
| `defaultValue` | `string?` | Fallback value used when the key is not found. Defaults to `""` if `null` |

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Key` | `string` | The configuration key |
| `DefaultValue` | `string` | The fallback value |

---

## Usage

Decorate static properties inside a `[Server]`-decorated class (or any auto-discovered class):

```csharp
[Server("ChatServer", 8443)]
public class ChatServer : WssServer
{
    [Config("WWW", "assets/client/dev")]
    private static string _staticContentPath { get; set; } = string.Empty;

    [Config("CERTIFICATE", "assets/tools/certificates/server.pfx")]
    private static string s_certPath { get; set; } = string.Empty;

    [Config("CERT_PASSWORD", "RootCA!SecureKey@Example2025Strong")]
    private static string s_certPassword { get; set; } = string.Empty;
}
```

At startup, LuciferCore resolves each key and sets the property value before calling any constructor.

---

## Remarks

- `[Config]` only applies to properties (`AttributeTargets.Property`).
- The property must be `static` — instance properties are not supported.
- If the key is not found and no `defaultValue` is provided, the property receives an empty string (`""`).
- Configuration resolution order and sources (environment variables, files, etc.) depend on the host configuration registered before `Lucifer.Run()` is called.
