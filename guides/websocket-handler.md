# WebSocket Handler

A **WebSocket Handler** processes incoming binary WebSocket frames dispatched by `Lucifer.Dispatch()`. Each handler method is mapped to a specific message type via `[WsMessage]`.

---

## The `[Handler]` Attribute

```csharp
[Handler("v1", "wss")]
internal class WssHandler : WssHandlerBase { ... }
```

| Parameter | Type | Description |
|---|---|---|
| `version` | `string` | API version namespace (e.g. `"v1"`) |
| `scope` | `string` | Protocol scope — use `"wss"` for WebSocket handlers |

---

## Handler Method Attributes

Each method in a handler class handles one message type. Stack the following attributes to control routing and security:

| Attribute | Description |
|---|---|
| `[WsMessage("name")]` | Maps this method to an incoming message type by name |
| `[Safe("")]` | Wraps the method in a safe execution context (catches and handles exceptions) |
| `[RateLimiter(limit, window)]` | Per-method rate limiting, in addition to any session-level limit |
| `[Authorize(role)]` | Requires the session to hold the specified role |

---

## Method Parameters

| Attribute | Type | Description |
|---|---|---|
| `[Session]` | Your session type | The active client session |
| `[Data]` | `PacketModel` | The deserialized incoming payload |

---

## Default Message Handler

Use `[WsMessage("Default")]` to catch any message that does not match a registered type:

```csharp
[WsMessage("Default")]
[Safe("")]
[RateLimiter(20, 1)]
[Authorize(UserRole.Guest)]
public void Default([Session] ChatSession session, [Data] PacketModel data)
    => throw new NotImplementedException();
```

---

## Full Example

```csharp
[Handler("v1", "wss")]
internal class WssHandler : WssHandlerBase
{
    public ConcurrentQueue<(byte[], long, long)> Messages = new();

    [WsMessage("GetMessage")]
    [Safe("")]
    [RateLimiter(10, 1)]
    [Authorize(UserRole.Guest)]
    public void GetMessage([Session] ChatSession session, [Data] PacketModel data)
    {
        foreach (var (buffer, offset, length) in Messages)
            session.SendBinaryAsync(buffer.AsSpan((int)offset, (int)length));
    }

    [WsMessage("ChatMessage")]
    [Safe("")]
    [RateLimiter(20, 1)]
    [Authorize(UserRole.Guest)]
    public void SendChat([Session] ChatSession session, [Data] PacketModel data)
    {
        using var _ = data;
        ((WssServer)session.Server).MulticastBinary(data.Buffer);
    }

    [WsMessage("Default")]
    [Safe("")]
    [RateLimiter(20, 1)]
    [Authorize(UserRole.Guest)]
    public void Default([Session] ChatSession session, [Data] PacketModel data)
        => throw new NotImplementedException();
}
```

> Always call `using var _ = data;` when you have finished reading `data.Buffer` to return the buffer to the pool and avoid memory leaks.
