# [Server]

**Namespace:** `LuciferCore.Attributes`

Marks a class as a server.

LuciferCore auto-discovers all classes with `[Server]` at startup and creates them with the given port.

> The server base type is not only `WssServer`/`HttpsServer`.  
> You can use other supported server bases in the transport stack (TCP/SSL/HTTP/WS/WSS and future extensions).  
> See API Reference for available server types.

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

| Parameter | Type | Meaning |
|---|---|---|
| `name` | `string` | Server name |
| `port` | `int` | Listening port |

---

## Properties

| Property | Type | Meaning |
|---|---|---|
| `Name` | `ByteString` | UTF-8 server name |
| `Port` | `int` | Listening port |

`ByteString` is a low-allocation UTF-8 type used by LuciferCore.

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

## Notes

- One `[Server]` per class (`AllowMultiple = false`).
- Attribute is not inherited (`Inherited = false`).
- Server name is used in logs and server management commands.
- Use `[Config]` in the same class for values like cert path and static folder.