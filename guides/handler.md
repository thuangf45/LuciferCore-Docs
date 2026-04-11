# Handler

A **Handler** processes incoming routable payloads dispatched by `Lucifer.Dispatch()`. Each handler method is mapped to a specific route via routing attributes — regardless of whether the data arrives over WebSocket, HTTP, or any custom protocol.

The routing system is **protocol-agnostic**. What matters is not the transport layer, but the type of data a method receives: a `PacketModel` for WebSocket binary frames, a `RequestModel` for HTTP requests, or any custom type that implements `IRoutable`. A single `RouteHandler` subclass can therefore serve multiple protocols simultaneously — the routing attribute and the `[Data]` parameter type are what differentiate one route from another.

---

## The `[Handler]` Attribute

```csharp
[Handler("v1", "wss")]
internal class ChatHandler : RouteHandler { ... }

[Handler("v1", "/api/user")]
internal class UserHandler : RouteHandler { ... }
```

| Parameter | Type     | Description                                                                 |
|-----------|----------|-----------------------------------------------------------------------------|
| `version` | `string` | API version namespace (e.g. `"v1"`)                                         |
| `scope`   | `string` | Protocol scope or base route — e.g. `"wss"`, `"ws"`, `"/api/user"` |

Both handlers extend the same `RouteHandler`. The distinction lies only in which routing attributes and data types the methods use — not in the base class.

---

## Routing Attributes

Decorate each handler method with the attribute that matches the incoming message type.

### WebSocket

| Attribute              | Description                                                        |
|------------------------|--------------------------------------------------------------------|
| `[WsMessage("name")]`  | Maps the method to an incoming WebSocket message type by name      |
| `[WsMessage("Default")]` | Catches any message that does not match a registered route       |

### HTTP

| Attribute                  | HTTP Verb  |
|----------------------------|------------|
| `[HttpGet("sub-route")]`    | `GET`      |
| `[HttpPost("sub-route")]`   | `POST`     |
| `[HttpPut("sub-route")]`    | `PUT`      |
| `[HttpDelete("sub-route")]` | `DELETE`   |
| `[HttpHead("sub-route")]`   | `HEAD`     |
| `[HttpOptions("sub-route")]`| `OPTIONS`  |
| `[HttpTrace("sub-route")]`  | `TRACE`    |

The string parameter is a sub-route appended to the handler's base scope. Pass an empty string (`""`) to match the base scope exactly.

---

## Method Parameters

| Attribute   | Expected Type                          | Description                          |
|-------------|----------------------------------------|--------------------------------------|
| `[Session]` | Any type extending `SessionTransport`  | The active client session            |
| `[Data]`    | Any type implementing `IRoutable`      | The incoming payload                 |

Both `PacketModel` (WebSocket) and `RequestModel` (HTTP) implement `IRoutable`. Custom data types can also participate in routing by implementing `IRoutable` directly — see [Custom IRoutable](#custom-iroutable) below.

---

## Security Attributes

The following attributes apply identically across all protocols and can be stacked on any handler method.

| Attribute                    | Description                                                                          |
|------------------------------|--------------------------------------------------------------------------------------|
| `[Safe("")]`                 | Wraps the method in a safe execution context that catches and handles exceptions     |
| `[RateLimiter(limit, window)]` | Applies per-method rate limiting in addition to any session-level limit            |
| `[Authorize(role)]`          | Requires the active session to hold the specified role before the method is invoked  |

---

## Examples

### WebSocket handler

```csharp
[Handler("v1", "wss")]
internal class ChatHandler : RouteHandler
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

> Always call `using var _ = data;` after reading `data.Buffer` to return the underlying buffer to the pool and avoid memory leaks.

### HTTP handler

```csharp
[Handler("v1", "/api/user")]
internal class UserHandler : RouteHandler
{
    [HttpGet("")]
    [Safe("")]
    [Authorize(UserRole.Guest)]
    [RateLimiter(100, 1)]
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [HttpPost("")]
    [Safe("")]
    [Authorize(UserRole.Guest)]
    [RateLimiter(100, 1)]
    protected void PostHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [HttpPut("")]
    [Safe("")]
    [Authorize(UserRole.Guest)]
    [RateLimiter(100, 1)]
    protected void PutHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();

    [HttpDelete("")]
    [Safe("")]
    [Authorize(UserRole.Guest)]
    [RateLimiter(100, 1)]
    protected void DeleteHandle([Data] RequestModel request, [Session] HttpsSession session)
        => throw new NotImplementedException();
}
```

### Custom IRoutable

Any data type can participate in routing by implementing `IRoutable`:

```csharp
public class MyPacket : IRoutable
{
    public string Action { get; set; } = "";
    public byte[] Payload { get; set; } = [];

    // Produces the route key: "MSG:/v1/myprotocol/{Action}"
    public ReadOnlySpan<byte> MethodRoute => "MSG"u8;
    public ReadOnlySpan<byte> UrlRoute    => Encoding.UTF8.GetBytes($"/v1/myprotocol/{Action}");
}

[Handler("v1", "myprotocol")]
internal class MyHandler : RouteHandler
{
    [WsMessage("Action")]
    [Safe("")]
    public void Handle([Data] MyPacket packet, [Session] MySession session) { ... }
}
```

---

## Remarks

- `RouteHandler` is the single base class for all handlers. The protocol-specific base classes (`WssHandlerBase`, `WsHandlerBase`, `HttpHandlerBase`, `HttpsHandlerBase`) no longer exist.
- Protocol differentiation is expressed through **routing attributes** (`[WsMessage]`, `[HttpGet]`, …) and the **`[Data]` parameter type** — not through the base class hierarchy.
- `[Session]` accepts any type that extends `SessionTransport`, giving each handler precise, type-safe access to its session without casting.
- Parameter type mismatches between `[Data]`/`[Session]` declarations and actual runtime types are caught at startup via `Debug.Assert`. A mismatched parameter receives `null` at runtime.
- To add cross-cutting logic such as IP filtering or session validation, override `CanHandle()` on your handler subclass.
- Query strings are stripped from `UrlRoute` automatically during route key construction — everything after `?` is ignored.
