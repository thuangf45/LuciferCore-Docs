# [Handler]

**Namespace:** `LuciferCore.Attributes`

Registers a class as a message or request handler. LuciferCore auto-discovers all `[Handler]`-decorated classes and wires them into the dispatch pipeline at startup.

The decorated class must extend `WssHandlerBase` (for WebSocket) or `HttpsHandlerBase` (for HTTP).

---

## Declaration

```csharp
[AttributeUsage(AttributeTargets.Class, AllowMultiple = false)]
public sealed class HandlerAttribute : Attribute
```

---

## Constructor

```csharp
public HandlerAttribute(string version = "v1", string prefix = "")
```

| Parameter | Type | Description |
|---|---|---|
| `version` | `string` | API version segment. Automatically prepended with `/` if missing. Defaults to `"v1"` |
| `prefix` | `string` | Route prefix for all methods in this handler. Automatically prepended with `/` if missing |

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Version` | `ByteString` | UTF-8 encoded version segment (e.g. `/v1`) |
| `Prefix` | `ByteString` | UTF-8 encoded route prefix (e.g. `/api/user`) |

---

## Usage

### WebSocket Handler

For WebSocket handlers, `prefix` is the protocol scope identifier (conventionally `"wss"`):

```csharp
[Handler("v1", "wss")]
internal class WssHandler : WssHandlerBase
{
    [WsMessage("ChatMessage")]
    public void SendChat([Session] ChatSession session, [Data] PacketModel data) { ... }
}
```

### HTTP Handler

For HTTP handlers, `prefix` is the base route path:

```csharp
[Handler("v1", "/api/user")]
internal class HttpsHandler : HttpsHandlerBase
{
    [HttpGet("")]
    protected void GetHandle([Data] RequestModel request, [Session] HttpsSession session) { ... }
}
```

---

## Routing Resolution

The full dispatch key is assembled from `version` + `prefix` + the method-level route:

```
/v1/api/user  +  [HttpGet("")]        →  GET  /v1/api/user
/v1/api/user  +  [HttpPost("/login")] →  POST /v1/api/user/login
/v1/wss       +  [WsMessage("Chat")]  →  MSG  /v1/wss/Chat
```

---

## Remarks

- Only one `[Handler]` attribute is allowed per class (`AllowMultiple = false`).
- The routing table is built once at startup into a lock-free structure — there is no per-request reflection.
- Handler classes are internal to the dispatch system and should not be instantiated manually.
