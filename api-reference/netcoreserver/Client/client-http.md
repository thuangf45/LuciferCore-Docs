# HttpClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

Plain HTTP client. Extends `TcpClient` with HTTP request building, response parsing, and a `Task`-based async request API with configurable timeout.

```csharp
public class HttpClient : TcpClient, IHttp
```

---

## Constructors

```csharp
public HttpClient(IPAddress address, int port)
public HttpClient(string address, int port)
public HttpClient(IPEndPoint endpoint)
public HttpClient(DnsEndPoint endpoint)
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Request` | `RequestModel` | The current request being built or sent |
| `Response` | `ResponseModel` | The most recently received response |

---

## Send API

### Fire-and-forget (sync + async)

```csharp
long SendRequest()
long SendRequest(RequestModel request)
long SendRequest(ReadOnlySpan<char> request)
long SendRequest(ReadOnlySpan<byte> request)

bool SendRequestAsync()
bool SendRequestAsync(RequestModel request)
bool SendRequestAsync(ReadOnlySpan<char> request)
bool SendRequestAsync(ReadOnlySpan<byte> request)
```

### Task-based with timeout (default: 1 second)

```csharp
Task<ResponseModel> SendRequest(TimeSpan? timeout = null)
Task<ResponseModel> SendRequest(RequestModel request, TimeSpan? timeout = null)
```

On timeout, the client disconnects and returns an empty `ResponseModel`.

### HTTP verb shortcuts

```csharp
Task<ResponseModel> SendGetRequest(string url, TimeSpan? timeout = null)
Task<ResponseModel> SendPostRequest(string url, string content, TimeSpan? timeout = null)
Task<ResponseModel> SendPutRequest(string url, string content, TimeSpan? timeout = null)
Task<ResponseModel> SendDeleteRequest(string url, TimeSpan? timeout = null)
Task<ResponseModel> SendHeadRequest(string url, TimeSpan? timeout = null)
Task<ResponseModel> SendOptionsRequest(string url, TimeSpan? timeout = null)
Task<ResponseModel> SendTraceRequest(string url, TimeSpan? timeout = null)
```

---

## Lifecycle Hooks

Override in your subclass:

| Method | When called |
|---|---|
| `OnConnectedOrHandshaked()` | After TCP connection is established — pending request is sent here |
| `OnReceivedResponseHeader(ResponseModel)` | Response headers parsed |
| `OnReceivedResponse(ResponseModel)` | **Primary hook** — full response received |
| `OnReceivedResponseError(ResponseModel, string)` | Response parse error |

---

## Usage

```csharp
var client = new HttpClient("api.example.com", 80);

// Task-based
var response = await client.SendGetRequest("/api/users");
Console.WriteLine(response.Status);

// Or extend for event-driven
public class MyClient : HttpClient
{
    public MyClient(string host) : base(host, 80) { }

    protected internal override void OnReceivedResponse(ResponseModel response)
        => Console.WriteLine($"Status: {response.Status}");
}
```

---

## Inherited API

Connect/disconnect, socket options, statistics, and base lifecycle hooks are inherited from `TcpClient` and `ClientTransport`. See [ClientTransport](transport-client.md).
