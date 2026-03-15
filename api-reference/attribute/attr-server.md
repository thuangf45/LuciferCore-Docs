# [Server]

**Namespace:** `LuciferCore.Attributes`

Registers a class as a network server. LuciferCore auto-discovers all `[Server]`-decorated classes at startup and instantiates them with the specified port.

The decorated class must extend `WssServer` or `HttpsServer`.

---

## Declaration

```csharp
[AttributeUsage(AttributeTargets.Class, Inherited = false, AllowMultiple = false)]
public class ServerAttribute : Attribute
```

---

## Constructor

```csharp
public ServerAttribute(string name, int port)
```

| Parameter | Type | Description |
|---|---|---|
| `name` | `string` | Human-readable identifier for this server instance |
| `port` | `int` | The port the server listens on |

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Name` | `ByteString` | UTF-8 encoded server name |
| `Port` | `int` | Listening port |

> `ByteString` is LuciferCore's zero-allocation UTF-8 string type. Names are encoded once at startup and never reallocated.

---

## Usage

```csharp
[Server("ChatServer", 8443)]
public class ChatServer : WssServer
{
    public ChatServer(int port) : this(CreateSslContext(), IPAddress.Any, port) { }

    protected override ChatSession CreateSession() => new(this);
}
```

---

## Remarks

- Only one `[Server]` attribute is allowed per class (`AllowMultiple = false`).
- The attribute is not inherited (`Inherited = false`) — subclasses are not automatically registered.
- The server name is used in console commands (`/start servers`, `/stop servers`) and log output.
- Use `[Config]` on static properties within the same class to bind configurable values such as certificate paths and static content directories. See [[Config]](attr-config.md).
