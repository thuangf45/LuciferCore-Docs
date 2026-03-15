# Configuring a Session

A **Session** represents a single connected client. It is the bridge between the raw socket layer and LuciferCore's dispatch pipeline. Every incoming message — whether WebSocket binary or HTTP request — passes through the session.

---

## The `[RateLimiter]` Attribute

Apply `[RateLimiter]` at the session class level to enforce a connection-wide rate limit before any handler is invoked.

```csharp
[RateLimiter(10, 1)]
public partial class ChatSession : WssSession { ... }
```

| Parameter | Type | Description |
|---|---|---|
| `limit` | `int` | Maximum number of messages allowed |
| `window` | `int` | Time window in seconds |

The example above allows a maximum of **10 messages per second** per connection.

---

## Forwarding to the Dispatcher

Override the appropriate `On*` methods and forward directly to `Lucifer.Dispatch()`. Do not add logic here — keep the session layer thin.

```csharp
// Forward incoming WebSocket binary frames
protected override void OnWsReceived(byte[] buffer, long offset, long size)
    => Lucifer.Dispatch(this, buffer, offset, size);

// Forward incoming HTTP requests
protected override void OnReceivedRequest(RequestModel request)
    => Lucifer.Dispatch(this, request);
```

`Lucifer.Dispatch()` routes each message through the zero-allocation pipeline to the correct handler method based on the message type or HTTP route.

---

## Full Example

```csharp
[RateLimiter(10, 1)]
public partial class ChatSession : WssSession
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public ChatSession(ChatServer server) : base(server) { }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected override void OnWsReceived(byte[] buffer, long offset, long size)
        => Lucifer.Dispatch(this, buffer, offset, size);

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected override void OnReceivedRequest(RequestModel request)
        => Lucifer.Dispatch(this, request);
}
```

> The `partial` modifier is required — LuciferCore's source generator may extend your session class at compile time.

---

## Next Steps

With the session configured, define the handlers that process each message type:
- [WebSocket Handler](websocket-handler.md)
- [HTTP Handler](http-handler.md)
