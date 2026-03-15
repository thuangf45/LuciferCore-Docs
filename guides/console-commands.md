# Console Commands

LuciferCore ships with an interactive console host that activates when `Lucifer.Run()` is called. It provides built-in lifecycle commands and a simple API for defining your own.

---

## Built-in Commands

| Command | Description |
|---|---|
| `/start host` | Start the host system |
| `/stop host` | Stop the host system |
| `/restart host` | Restart the host system |
| `/start servers` | Start all `[Server]`-decorated instances |
| `/stop servers` | Stop all `[Server]`-decorated instances |
| `/restart servers` | Restart all `[Server]`-decorated instances |
| `/start managers` | Start all `[Manager]`-decorated instances |
| `/stop managers` | Stop all `[Manager]`-decorated instances |
| `/restart managers` | Restart all `[Manager]`-decorated instances |

---

## Defining Custom Commands

Use `[ConsoleCommand]` on any static method to register a custom command. LuciferCore auto-discovers these at startup alongside servers, handlers, and managers.

```csharp
[ConsoleCommand("/start proxy", "Start proxy")]
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private static void CmdStartProxy() => Start();

[ConsoleCommand("/stop proxy", "Stop proxy")]
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private static void CmdStopProxy() => Stop();
```

### Attribute Parameters

| Parameter | Type | Description |
|---|---|---|
| `command` | `string` | The exact text the user types (e.g. `"/start proxy"`) |
| `description` | `string` | Short description shown in the help listing |

---

## Tips

- Command strings are case-sensitive by default. Use a consistent naming convention such as `/verb noun`.
- Keep command handler methods thin — delegate to service methods (`Start()`, `Stop()`) rather than embedding logic directly.
- Custom commands are discovered from any class in your assembly — no registration needed.
