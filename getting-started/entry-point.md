# Entry Point

The entry point for any LuciferCore application is `Program.cs`. It is intentionally minimal — the framework handles all discovery and orchestration automatically.

---

## Minimal Entry Point

```csharp
using LuciferCore.Main;

Lucifer.Run();
```

This single call:
- Auto-discovers all classes decorated with `[Server]`, `[Handler]`, and `[Manager]`
- Wires up async dispatch pipelines
- Starts all servers and managers
- Launches the interactive console host

---

---

## Built-in Console Commands

Once `Lucifer.Run()` is called, the console host is active with the following built-in commands:

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
| `/start services` | Start all `[Service]`-decorated instances |
| `/stop services` | Stop all `[Service]`-decorated instances |
| `/restart services` | Restart all `[Service]`-decorated instances |

---

## Defining Custom Console Commands

Extend the console host with your own commands using `[ConsoleCommand]`:

```csharp
[ConsoleCommand("/start proxy", "Start proxy")]
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private static void CmdStartProxy() => Start();

[ConsoleCommand("/stop proxy", "Stop proxy")]
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private static void CmdStopProxy() => Stop();
```

The attribute takes two parameters:
- **Command string** — the text the user types in the console (e.g. `/start proxy`)
- **Description** — shown in the help listing

Custom commands are auto-discovered alongside servers, handlers, and managers at startup.
