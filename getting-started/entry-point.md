# Entry Point

`Program.cs` is the entry point of a LuciferCore app.  
It stays minimal because LuciferCore auto-discovers and wires everything for you.

---

## Minimal `Program.cs`

```csharp
using LuciferCore.Main;

Lucifer.CMD("/run"u8);
```

This one line will:

- Discover classes with `[Server]`, `[Handler]`, `[Middleware]`, `[Manager]`, `[Route]`
- Build async dispatch pipelines
- Start servers and managers
- Start the interactive console host

---

## Built-in Console Commands

After `Lucifer.CMD("/run"u8)`, these commands are available:

| Command | Description |
|---|---|
| `/start host` | Start host system |
| `/stop host` | Stop host system |
| `/restart host` | Restart host system |
| `/start servers` | Start all servers marked with `[Server]` |
| `/stop servers` | Stop all servers marked with `[Server]` |
| `/restart servers` | Restart all servers marked with `[Server]` |
| `/start managers` | Start all managers marked with `[Manager]` |
| `/stop managers` | Stop all managers marked with `[Manager]` |
| `/restart managers` | Restart all managers marked with `[Manager]` |

---

## Add Custom Console Commands

Use `[ConsoleCommand]` to add your own commands:

```csharp
[ConsoleCommand("/start proxy", "Start proxy")]
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private static void CmdStartProxy() => Start();

[ConsoleCommand("/stop proxy", "Stop proxy")]
[MethodImpl(MethodImplOptions.AggressiveInlining)]
private static void CmdStopProxy() => Stop();
```

`[ConsoleCommand]` has 2 parameters:

- **Command string**: what the user types (example: `/start proxy`)
- **Description**: shown in help output

Custom commands are also auto-discovered at startup.