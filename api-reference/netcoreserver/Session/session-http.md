# HttpSession

**Namespace:** `LuciferCore.NetCoreServer.Session`

`HttpSession` is the plain HTTP session class.  
It extends `TcpSession` and handles HTTP request parsing + response sending.

```csharp
public class HttpSession : TcpSession
```

---

## Constructor

```csharp
public HttpSession(HttpServer server)
```

Created by `HttpServer.CreateSession()`.  
It keeps a reference to the server `Cache`.

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Cache` | `FileCache` | Static file cache from the parent server |
| `Mapping` | `Utf8Map<ByteString>` | URL mapping from `HttpServer.Mapping` |

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
public class MySession : HttpSession
{
    public MySession(HttpServer server) : base(server) { }

    protected internal override void OnReceivedRequest(RequestModel request)
    {
        using var response = Lucifer.Rent<ResponseModel>();
        SendResponseAsync(response.MakeGetResponse("Hello"u8, "text/plain"u8));
    }
}
```

---

## Inherited API

`HttpSession` adds HTTP request/response behavior.  
Other APIs are inherited from `TcpSession` / `SessionTransport`, including:

- send/receive base methods
- disconnect/dispose
- lifecycle hooks/events
- metrics/options access