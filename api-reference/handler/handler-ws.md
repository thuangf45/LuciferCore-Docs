# WebSocket Handler Bases

**Namespace:** `LuciferCore.Handler.Transports`

LuciferCore provides two WebSocket handler base classes depending on whether the connection is encrypted:

| Class | Protocol | Session Type |
|---|---|---|
| `WsHandlerBase` | WebSocket (unencrypted) | `WsSession` |
| `WssHandlerBase` | WebSocket Secure (TLS) | `WssSession` |

Both classes share identical behavior and API surface. The only difference is the session type they bind to and the error response path they use internally.

---

## WssHandlerBase

```csharp
public abstract class WssHandlerBase : HandlerBase, IHandler<PacketModel, WssSession>
```

Base class for all WSS (WebSocket over TLS) handlers. Extend this class when your server is declared as `WssServer`.

### WsHandlerBase

```csharp
public abstract class WsHandlerBase : HandlerBase, IHandler<PacketModel, WsSession>
```

Base class for unencrypted WebSocket handlers. Extend this when your server is declared as `WsServer`.

---

## Route Key Format

Both bases build the dispatch key from the incoming `PacketModel.UrlView`:

```
MSG:/{version}/{prefix}/{message-type}

// Example:
MSG:/v1/wss/ChatMessage
MSG:/v1/wss/Default
```

The `MSG` prefix distinguishes WebSocket routes from HTTP routes in the shared routing table.

---

## Default Handler

Both base classes declare a built-in default handler that catches any message type with no matching route:

```csharp
[Safe]
[WsMessage("Default")]
[RateLimiter(1000, 1)]
public virtual void DefaultHandler([Data] PacketModel data, [Session] WssSession session)
    => throw new NotImplementedException();
```

Override `DefaultHandler` in your subclass to provide custom fallback behavior instead of `NotImplementedException`:

```csharp
[Handler("v1", "wss")]
internal class WssHandler : WssHandlerBase
{
    // Custom fallback — log and ignore unknown messages
    public override void DefaultHandler([Data] PacketModel data, [Session] WssSession session)
    {
        Lucifer.Log(this, $"Unknown message received.");
    }

    [WsMessage("ChatMessage")]
    [Safe("")]
    [RateLimiter(20, 1)]
    [Authorize(UserRole.Guest)]
    public void SendChat([Session] WssSession session, [Data] PacketModel data)
    {
        using var _ = data;
        ((WssServer)session.Server).MulticastBinary(data.Buffer);
    }
}
```

---

## Error Responses

When a request fails (unrouted, rate-limited, or unauthorized), both bases send an error back to the client. The encoding depends on the connection state:

| Connection State | Error Response |
|---|---|
| WebSocket handshake complete (`IsWebSocket == true`) | Binary `PacketModel` frame sent to the path `/v1/wss/Error` (or `/v1/ws/Error`) |
| HTTP upgrade phase (before handshake) | Standard HTTP error response via `MakeErrorResponse(code, message)` |

---

## Extending

### Minimal handler

```csharp
[Handler("v1", "wss")]
internal class WssHandler : WssHandlerBase
{
    [WsMessage("Ping")]
    [Safe("")]
    [RateLimiter(60, 1)]
    [Authorize(UserRole.Guest)]
    public void Ping([Session] WssSession session, [Data] PacketModel data)
    {
        // respond with pong
    }
}
```

### Handler with shared state

Fields declared on the handler class are shared across all dispatch invocations on that handler instance. Use thread-safe collections for mutable shared state:

```csharp
[Handler("v1", "wss")]
internal class WssHandler : WssHandlerBase
{
    public ConcurrentQueue<(byte[], long, long)> Messages = new();

    [WsMessage("GetMessage")]
    [Safe("")]
    [RateLimiter(10, 1)]
    [Authorize(UserRole.Guest)]
    public void GetMessage([Session] WssSession session, [Data] PacketModel data)
    {
        foreach (var (buffer, offset, length) in Messages)
            session.SendBinaryAsync(buffer.AsSpan((int)offset, (int)length));
    }
}
```

---

## Remarks

- Always call `using var _ = data;` after you have finished reading `data.Buffer` to return the buffer to the pool.
- Do not call `Handle()` directly — it is invoked by `Lucifer.Dispatch()` through the pipeline.
- The `DefaultHandler` built-in rate limit is `1000 per second`. Override the method and redeclare `[RateLimiter]` if you need a different limit.
- `Handle()` is `virtual` — you can override it to intercept all dispatch for a handler before route resolution.
