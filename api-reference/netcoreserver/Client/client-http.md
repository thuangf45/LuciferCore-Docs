# HttpClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

Plain HTTP client. Extends `TcpClient` with HTTP request building and incremental response parsing.

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
| `Request` | `RequestModel` | The current request, lazily rented from the pool (`Lucifer.Rent<RequestModel>()`) on first access |
| `Response` | `ResponseModel` | The current response, lazily rented from the pool on first access |

---

## Send API

```csharp
long SendRequest()
long SendRequest(RequestModel request)

bool SendRequestAsync()
bool SendRequestAsync(RequestModel request)
```

`SendRequest()` / `SendRequestAsync()` send the client's current `Request`. Both overloads that take an explicit `RequestModel` automatically add a `Host` header (set to `Address`) before sending — you don't need to set it yourself.

There is no built-in `Task`-based await API, timeout handling, or HTTP-verb shortcut methods (`SendGetRequest`, etc.) in this class — requests are fire-and-forget; use the lifecycle hooks below to react to the response.

---

## Lifecycle Hooks

Override in your subclass:

| Method | When called |
|---|---|
| `OnReceivedResponseHeader(ResponseModel)` | Response headers parsed. Default implementation is empty |
| `OnReceivedResponse(ResponseModel)` | **Primary hook** — full response body received. Default implementation is empty |
| `OnReceivedResponseError(ResponseModel, string)` | Header or body parse error — the client disconnects immediately after |

---

## Receive / Disconnect Behavior

- `OnReceived` parses incrementally: header first (fires `OnReceivedResponseHeader`), then body (fires `OnReceivedResponse` and clears `Response` for reuse). A parse error at either stage fires `OnReceivedResponseError`, clears `Response`, and disconnects.
- `OnDisconnected` flushes a response whose body was still pending (calls `OnReceivedResponse` with the partial response), then disposes and releases both `Request` and `Response` back to the pool before calling `base.OnDisconnected()`.

---

## Usage

```csharp
public class MyClient : HttpClient
{
    public MyClient(string host) : base(host, 80) { }

    protected internal override void OnReceivedResponse(ResponseModel response)
        => Console.WriteLine($"Status: {response.Status}");
}

var client = new MyClient("api.example.com");
client.Connect();
client.SendRequest(); // build up client.Request beforehand as needed
```

---

## Inherited API

Connect/disconnect, socket options, statistics, and base lifecycle hooks are inherited from `TcpClient`.

---