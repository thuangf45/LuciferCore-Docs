# HttpsClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

`HttpsClient` is the secure HTTP client class.  
It extends `SslClient` and adds HTTP request/response handling over TLS.

```csharp
public class HttpsClient : SslClient
```

---

## Constructors

```csharp
public HttpsClient(SslContext context, string host)
public HttpsClient(SslContext context, DnsEndPoint endpoint)
public HttpsClient(SslContext context, IPAddress address, int port)
public HttpsClient(SslContext context, string address, int port)
public HttpsClient(SslContext context, IPEndPoint endpoint)
public HttpsClient(SslContext context, EndPoint endpoint, string address, int port)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Request` | `RequestModel` | Current request object |
| `Response` | `ResponseModel` | Current response object |

---

## Send API

```csharp
long SendRequest()
long SendRequest(RequestModel request)

bool SendRequestAsync()
bool SendRequestAsync(RequestModel request)
```

- Sends current request or a provided request
- `Host` header is set automatically from `Address`
- `SendRequestAsync(...)` is non-blocking

---

## Main hooks to override

| Method | When called |
|---|---|
| `OnReceivedResponseHeader(ResponseModel)` | Response header is parsed |
| `OnReceivedResponse(ResponseModel)` | Full response is ready (main handler) |
| `OnReceivedResponseError(ResponseModel, string)` | Response parse error |

`OnReceivedResponse(...)` is the main method for response handling.

---

## Custom client example

```csharp
public class MyHttpsClient : HttpsClient
{
    public MyHttpsClient(SslContext context, string host) : base(context, host) { }

    protected internal override void OnReceivedResponse(ResponseModel response)
        => Console.WriteLine($"Status: {response.Status}");
}

var client = new MyHttpsClient(context, "api.example.com");
client.Connect();
client.SendRequest();
```

---

## Inherited API

`HttpsClient` adds HTTPS request/response behavior.  
Other APIs are inherited from `SslClient` / `ClientTransport`, including:

- TLS handshake/connect/disconnect
- send/receive base methods
- lifecycle hooks/events
- metrics/options access
- `Dispose()`