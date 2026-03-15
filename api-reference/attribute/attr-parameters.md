# Parameter Attributes

**Namespace:** `LuciferCore.Attributes`

Parameter attributes mark handler method parameters so the dispatch pipeline knows how to inject the correct values. They carry no runtime cost beyond startup wiring.

There are two parameter attributes: `[Session]` and `[Data]`.

---

## `[Session]`

Marks a method parameter as the active client session. The dispatch pipeline injects the session instance that received the incoming message or request.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Parameter)]
public class SessionAttribute : Attribute { }
```

### Usage

Declare the parameter with your concrete session type (not the base class) so you have direct access to session-specific members:

```csharp
// WebSocket handler
public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }

// HTTP handler
protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
```

---

## `[Data]`

Marks a method parameter as the incoming payload. The dispatch pipeline injects the deserialized message or request model that was received.

### Declaration

```csharp
[AttributeUsage(AttributeTargets.Parameter)]
public class DataAttribute : Attribute { }
```

### Usage

The parameter type depends on the handler protocol:

| Protocol | Parameter Type | Description |
|---|---|---|
| WebSocket | `PacketModel` | Binary WebSocket frame payload |
| HTTP | `RequestModel` | Full HTTP request including headers, body, and route |

```csharp
// WebSocket handler — PacketModel
public void SendChat([Session] ChatSession session, [Data] PacketModel data)
{
    using var _ = data; // return buffer to pool after use
    ((WssServer)session.Server).MulticastBinary(data.Buffer);
}

// HTTP handler — RequestModel
protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
```

---

## Parameter Order

Parameter order in handler method signatures is **flexible** — the dispatch pipeline identifies each parameter by its attribute, not its position. Both orderings are valid:

```csharp
// Session first, then Data
public void Handle([Session] ChatSession session, [Data] PacketModel data) { ... }

// Data first, then Session
public void Handle([Data] PacketModel data, [Session] ChatSession session) { ... }
```

---

## Remarks

- Both attributes target `AttributeTargets.Parameter` only and have no effect on other targets.
- Always call `using var _ = data;` on `PacketModel` when you have finished reading `data.Buffer`. This returns the underlying buffer to the pool and prevents memory leaks.
- Use your concrete session type (e.g. `ChatSession`) rather than the base type (e.g. `WssSession`) to avoid unnecessary casts inside the handler.
