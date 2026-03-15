# [RateLimiter]

**Namespace:** `LuciferCore.Attributes`

Enforces a request rate limit on a method or class. When the limit is exceeded, the incoming message or request is dropped before the handler body executes — at zero allocation cost.

Can be applied at two levels:

- **Class level** (on a `Session`) — connection-wide limit applied to every incoming message from that client.
- **Method level** (on a handler method) — per-message-type limit applied after the connection-wide check.

---

## Declaration

```csharp
[AttributeUsage(AttributeTargets.Method | AttributeTargets.Class, Inherited = true, AllowMultiple = false)]
public class RateLimiterAttribute : Attribute
```

---

## Constructor

```csharp
public RateLimiterAttribute(int limit = 1, int periodSeconds = 60)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `limit` | `int` | `1` | Maximum number of requests allowed within the time window |
| `periodSeconds` | `int` | `60` | Duration of the time window in seconds |

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Limit` | `int` | Maximum allowed requests per window |
| `PeriodSeconds` | `int` | Time window duration in seconds |

---

## Usage

### Session-level (connection-wide)

Applied to the session class. Limits all incoming messages from a single connection regardless of message type.

```csharp
[RateLimiter(10, 1)] // max 10 messages per second per connection
public partial class ChatSession : WssSession { ... }
```

### Method-level (per message type)

Applied to individual handler methods. Evaluated after the session-level check passes.

```csharp
[WsMessage("ChatMessage")]
[RateLimiter(20, 1)] // max 20 of this message type per second
public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }
```

```csharp
[HttpPost("")]
[RateLimiter(100, 1)] // max 100 POSTs per second
protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
```

### Stacking both levels

Both limits are enforced independently. A request must pass the session-level check **and** the method-level check:

```csharp
[RateLimiter(50, 1)]          // session: max 50 total messages/sec
public partial class ChatSession : WssSession { ... }

[WsMessage("ChatMessage")]
[RateLimiter(20, 1)]          // method: max 20 ChatMessage/sec
public void SendChat(...) { ... }
```

---

## Remarks

- `Inherited = true` — subclasses and overriding methods inherit the rate limit unless they declare their own.
- `AllowMultiple = false` — only one `[RateLimiter]` per target. Declare at the most specific level needed.
- Rate limiting is enforced in the lock-free dispatch pipeline before any handler allocation occurs.
- Exceeded requests are silently dropped. To send an error response on limit exceeded, pair with `[Safe]` and handle the drop in the safe context.
