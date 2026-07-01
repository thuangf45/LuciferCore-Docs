# HttpClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

`HttpClient` is the plain HTTP client class.  
It extends `TcpClient` and adds HTTP request/response handling.

```csharp
public class HttpClient : TcpClient
```

---

## Constructors

```csharp
public HttpClient(string host)
public HttpClient(DnsEndPoint endpoint)
public HttpClient(IPAddress address, int port)
public HttpClient(string address, int port)
public HttpClient(IPEndPoint endpoint)
public HttpClient(EndPoint endpoint, string address, int port)
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

- Sends the current request or a provided request
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
public class MyClient : HttpClient
{
    public MyClient(string host) : base(host, 80) { }

    protected internal override void OnReceivedResponse(ResponseModel response)
        => Console.WriteLine($"Status: {response.Status}");
}

var client = new MyClient("api.example.com");
client.Connect();
client.SendRequest();
```

---

## Inherited API

`HttpClient` adds HTTP request/response behavior.  
Other APIs are inherited from `TcpClient`, including:

- connect/reconnect/disconnect
- send/receive base methods
- lifecycle hooks/events
- metrics/options access
- `Dispose()`