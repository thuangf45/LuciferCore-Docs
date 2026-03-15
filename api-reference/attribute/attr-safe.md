# [Safe]

**Namespace:** `LuciferCore.Attributes`

Wraps a handler method in a safe execution context. Any unhandled exception thrown inside the method is caught by the dispatch pipeline, preventing it from propagating up and crashing the server or session.

An optional error message can be returned to the caller when the method fails.

---

## Declaration

```csharp
[AttributeUsage(AttributeTargets.Method)]
public class SafeAttribute : Attribute
```

---

## Constructor

```csharp
public SafeAttribute(string? message = null)
```

| Parameter | Type | Default | Description |
|---|---|---|---|
| `message` | `string?` | `null` | Optional error message sent back to the caller on failure. Pass `""` for a silent fail with no response |

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Message` | `ByteString` | UTF-8 encoded error message. Empty if none was provided |

---

## Usage

### Silent fail (no error response)

The method is protected from crashing, and no message is returned to the client on failure:

```csharp
[WsMessage("ChatMessage")]
[Safe("")]
public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }
```

### Fail with error message

On failure, the specified message is returned to the caller:

```csharp
[HttpPost("")]
[Safe("An error occurred. Please try again.")]
protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
```

### Typical full stack

In practice, `[Safe]` is always stacked with `[RateLimiter]` and `[Authorize]`:

```csharp
[WsMessage("GetMessage")]
[Safe("")]
[RateLimiter(10, 1)]
[Authorize(UserRole.Guest)]
public void GetMessage([Session] ChatSession session, [Data] PacketModel data) { ... }
```

---

## Execution Order

Attributes are evaluated in the following order by the dispatch pipeline:

```
[RateLimiter] → [Authorize] → [Safe] → handler body
```

`[Safe]` is the innermost guard — it wraps the actual method execution. Exceptions thrown **before** `[Safe]` (e.g. during rate limit checks) are handled by the pipeline itself.

---

## Remarks

- `[Safe]` only applies to methods (`AttributeTargets.Method`).
- Without `[Safe]`, any unhandled exception in a handler method bubbles up to the session and may terminate the connection.
- The error message is encoded as a `ByteString` once at startup and reused without allocation on every failure.
- For development, omitting `[Safe]` can be useful to surface exceptions clearly. Add it before deploying to production.
