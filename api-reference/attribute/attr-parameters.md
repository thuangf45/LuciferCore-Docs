# Parameter Attributes

**Namespace:** `LuciferCore.Attributes`

Parameter attributes tell the dispatcher what to inject into handler method parameters.

Two attributes:
- `[Session]`
- `[Data]`

---

## `[Session]`

Marks a parameter as the current client session.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Parameter)]
public class SessionAttribute : Attribute { }
```

### Usage

Use your concrete session type for easier access to custom fields/methods.

```csharp
public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }

protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
```

---

## `[Data]`

Marks a parameter as incoming payload.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Parameter)]
public class DataAttribute : Attribute { }
```

### Usage

`[Data]` type depends on route/protocol:

| Input type | Data type | Meaning |
|---|---|---|
| Message route (WS/WSS/custom) | `PacketModel` or custom `IRoutable` | Message payload |
| HTTP route | `RequestModel` | HTTP request |

```csharp
public void SendChat([Session] ChatSession session, [Data] PacketModel data)
{
    using var _ = data;
    ((WssServer)session.Server).MulticastBinary(data.Buffer);
}

protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
```

> For `PacketModel`, use `using var _ = data;` after reading `data.Buffer` to return buffer to pool.

---

## Parameter order

Order is flexible.  
Dispatcher uses attributes, not position.

```csharp
public void Handle([Session] ChatSession session, [Data] PacketModel data) { ... }

public void Handle([Data] PacketModel data, [Session] ChatSession session) { ... }
```

---

## Notes

- Both attributes are for parameters only.
- Use concrete session type (example: `ChatSession`) instead of generic base type when possible.
- `[Data]` is extensible: any custom type implementing `IRoutable` can be injected.