# Handler

A **Handler** processes incoming data.

`Lucifer.Dispatch()` sends data to the correct handler method.

A handler can work with many protocols (WebSocket, HTTP, custom protocol).  
Routing depends on:
- routing attribute (`[Message]`, `[HttpGet]`, ...)
- `[Data]` type (`PacketModel`, `RequestModel`, or custom `IRoutable`)

It does **not** depend on different handler base classes.

---

## `[Handler]` attribute

```csharp
[Handler("v1", "wss")]
internal class ChatHandler : RouteHandler { ... }

[Handler("v1", "/api/user")]
internal class UserHandler : RouteHandler { ... }
```

| Parameter | Type | Meaning |
|---|---|---|
| `version` | `string` | API version (example: `"v1"`) |
| `scope` | `string` | Base scope/route (example: `"wss"`, `"/api/user"`) |

---

## Routing attributes

### WebSocket routes

| Attribute | Meaning |
|---|---|
| `[Message("name")]` | Match WebSocket message by name |
| `[Message("Default")]` | Fallback route when no route matches |

### HTTP routes

| Attribute | HTTP verb |
|---|---|
| `[HttpGet("sub-route")]` | GET |
| `[HttpPost("sub-route")]` | POST |
| `[HttpPut("sub-route")]` | PUT |
| `[HttpDelete("sub-route")]` | DELETE |
| `[HttpHead("sub-route")]` | HEAD |
| `[HttpOptions("sub-route")]` | OPTIONS |
| `[HttpTrace("sub-route")]` | TRACE |

`"sub-route"` is appended to handler scope.  
Use `""` to match the base scope exactly.

---

## Method parameters

| Attribute | Type | Meaning |
|---|---|---|
| `[Session]` | Any type from `SessionTransport` | Current client session |
| `[Data]` | Any type from `IRoutable` | Incoming payload |

Examples:
- `PacketModel` (WebSocket)
- `RequestModel` (HTTP)
- your own custom type implementing `IRoutable`

---

## Security attributes

| Attribute | Meaning |
|---|---|
| `[RateLimiter(limit, window)]` | Limit request rate for this method |
| `[Authorize(role)]` | Require role before method runs |

These are built-in options.  
For more control, use **middleware**.

### Use middleware for advanced security

You can attach one or many middlewares with `[UseMiddleware(...)]`.

```csharp
[HttpGet("")]
[UseMiddleware("AuthGuard")]
[UseMiddleware("IpFilter")]
[Authorize(UserRole.Guest)]
public void GetProfile([Session] HttpsSession session, [Data] RequestModel request)
{
    // handler logic
}
```

- You can chain multiple middlewares.
- You can create your own middleware type.
- See **API Reference** for built-in middleware options.
- See **Middleware Guide** to build custom middleware.

### You can also extend routing

Routing is extensible too:
- You can define custom route labels/attributes.
- You can define custom handler patterns.
- You can build your own route style for custom protocols.

So in LuciferCore, both **middleware** and **routing** are open for extension.

---

## Examples

### WebSocket handler

```csharp
[Handler("v1", "wss")]
internal class ChatHandler : RouteHandler
{
    public ConcurrentQueue<(byte[], long, long)> Messages = new();

    [Message("GetMessage")]
    [Authorize(UserRole.Guest)]
    public void GetMessage([Session] ChatSession session, [Data] PacketModel data)
    {
        foreach (var (buffer, offset, length) in Messages)
            session.SendBinaryAsync(buffer.AsSpan((int)offset, (int)length));
    }

    [Message("ChatMessage")]
    [Authorize(UserRole.Guest)]
    public void SendChat([Session] ChatSession session, [Data] PacketModel data)
    {
        using var _ = data;
        ((WssServer)session.Server).MulticastBinary(data.Buffer);
    }

    [Message("Default")]
    [Authorize(UserRole.Guest)]
    public void Default([Session] ChatSession session, [Data] PacketModel data)
        => throw new NotImplementedException();
}
```

> Use `using var _ = data;` after using `data.Buffer` to return buffer to pool.

### HTTP handler

```csharp
[Handler("v1", "/api/user")]
internal class UserHandler : RouteHandler
{
    [HttpGet("")]
    [Authorize(UserRole.Guest)]
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [HttpPost("")]
    [Authorize(UserRole.Guest)]
    protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [HttpPut("")]
    [Authorize(UserRole.Guest)]
    protected void PutHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [HttpDelete("")]
    [Authorize(UserRole.Guest)]
    protected void DeleteHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();
}
```

### Custom `IRoutable`

```csharp
public class MyPacket : IRoutable
{
    public string Action { get; set; } = "";
    public byte[] Payload { get; set; } = [];

    public ReadOnlySpan<byte> MethodRoute => "MSG"u8;
    public ReadOnlySpan<byte> UrlRoute => Encoding.UTF8.GetBytes($"/v1/myprotocol/{Action}");
}

[Handler("v1", "myprotocol")]
internal class MyHandler : RouteHandler
{
    [Message("Action")]
    public void Handle([Data] MyPacket packet, [Session] MySession session) { ... }
}
```

---

## Notes

- Use only `RouteHandler` as handler base class.
- Protocol choice comes from route attribute + `[Data]` type.
- `[Session]` can be any session type from `SessionTransport`.
- If `[Data]`/`[Session]` type is wrong, startup assertion catches it (and runtime value may be `null`).
- For cross-cutting checks (IP, session state, etc.), override `CanHandle()`.
- Query string is ignored in route key (`?x=...` part is removed).
```