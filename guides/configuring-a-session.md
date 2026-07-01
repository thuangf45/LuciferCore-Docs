# Configuring a Session

A **Session** represents one connected client for a specific server stack.

It bridges:
- low-level transport events (TCP/SSL/HTTP/WS)
- LuciferCore routing and dispatch

All inbound data goes through the session first.

---

## Sessions are protocol-stack dependent

LuciferCore supports multiple session types, not only `WssSession`.

Depending on your selected server base, session can be based on:
- TCP
- SSL
- HTTP / HTTPS
- WS / WSS
- and future protocol sessions added later

> Choose session base type from API Reference (NetCoreServer hierarchy + LuciferCore extensions).

---

## Layer behavior and protocol transition

With higher-layer sessions (for example WS/WSS):
- Before WebSocket handshake, traffic is still HTTP/HTTPS.
- After handshake, traffic becomes WebSocket frames.

You can also hook and inspect data at different layers (transport/TCP/SSL/HTTP/WS), based on where your logic belongs.

---

## `[RateLimiter]` attribute

Use `[RateLimiter]` on session class to apply per-connection limits before handler execution.

```csharp
[RateLimiter(10, 1)]
public partial class ChatSession : WssSession { ... }
```

| Parameter | Type | Description |
|---|---|---|
| `limit` | `int` | Max allowed messages/events |
| `window` | `int` | Time window in seconds |

Example: `[RateLimiter(10, 1)]` = up to **10 messages per second** per connection.

---

## Forward data to router

Override receive methods for your session type and forward routable requests to `Lucifer.Route(...)`.

Keep session code thin.  
Business logic should stay in handlers/middlewares/managers.

```csharp
protected override void OnWsReceived(byte[] buffer, long offset, long size)
{
    // Optional: protocol-level handling
}

protected override void OnReceivedRequest(RequestModel request)
    => Lucifer.Route(request, this);
```

`Lucifer.Route(...)` dispatches to the correct endpoint/handler.

---

## Example (WSS session)

```csharp
[RateLimiter(10, 1)]
public partial class ChatSession : WssSession
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public ChatSession(ChatServer server) : base(server) { }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected override void OnWsReceived(byte[] buffer, long offset, long size)
    {
        // Optional: process websocket message
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected override void OnReceivedRequest(RequestModel request)
        => Lucifer.Route(request, this);
}
```

> `partial` is required because LuciferCore source generators may extend your session class.

---

## Keep an eye on new protocol docs

LuciferCore can expand with more server/session types over time.  
Follow the docs and API Reference for newly supported protocol stacks.