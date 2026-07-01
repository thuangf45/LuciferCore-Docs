# HttpsSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

`HttpsSession` is the secure HTTP session class.  
It extends `SslSession` and handles HTTP request parsing + response sending over TLS.

```csharp
public class HttpsSession : SslSession
```

---

## Constructor

```csharp
public HttpsSession(HttpsServer server)
```

Created by `HttpsServer.CreateSession()`.  
It keeps a reference to the server `Cache`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Cache` | `FileCache` | Static file cache from the parent server |
| `Mapping` | `Utf8Map<ByteString>` | URL mapping from `HttpsServer.Mapping` |

---

## Response API

```csharp
long SendResponse(ResponseModel response)
bool SendResponseAsync(ResponseModel response)
```

- `SendResponse(...)`: sync send
- `SendResponseAsync(...)`: async send (recommended for normal request flow)

---

## Main hooks to override

| Method | When called |
|---|---|
| `OnReceivedRequestHeader(RequestModel)` | Request header is parsed |
| `OnReceivedRequest(RequestModel)` | Full request is ready (main handler) |
| `OnReceivedRequestError(RequestModel, string)` | Request parse error |
| `OnReceivedCachedRequest(RequestModel, byte[])` | Static cache hit |

`OnReceivedRequest(...)` is the main method to handle your API/business logic.

---

## Request flow (simple)

When a full request is ready:

1. check overload/allow rules
2. if `GET` and not `/api`, try static cache
3. cache hit → call `OnReceivedCachedRequest(...)`
4. otherwise → call `OnReceivedRequest(...)`

---

## Custom session example

```csharp
public class ChatSession : HttpsSession
{
    public ChatSession(HttpsServer server) : base(server) { }

    protected internal override void OnReceivedRequest(RequestModel request)
    {
        using var response = Lucifer.Rent<ResponseModel>();
        SendResponseAsync(response.MakeGetResponse("Hello"u8, "text/plain"u8));
    }
}
```

---

## Inherited API

`HttpsSession` adds HTTPS request/response behavior.  
Other APIs are inherited from `SslSession`, including:

- send/receive base methods
- disconnect/dispose
- lifecycle hooks/events
- metrics/options access