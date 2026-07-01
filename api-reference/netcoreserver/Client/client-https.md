# HttpsClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

Secure HTTPS client. Extends `SslClient` with HTTP request building and incremental response parsing over TLS.

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

`SendRequest()` / `SendRequestAsync()` send the client's current `Request`. Both overloads that take an explicit `RequestModel` automatically add a `Host` header (set to `Address`) before sending.

There is no built-in `Task`-based await API, timeout handling, or HTTP-verb shortcut methods (`SendGetRequest`, etc.) in this class — identical send surface to `HttpClient`, just carried over TLS via `SslClient`.

---

## Lifecycle Hooks

Override in your subclass:

| Method | When called |
|---|---|
| `OnReceivedResponseHeader(ResponseModel)` | Response headers parsed. Default implementation is empty |
| `OnReceivedResponse(ResponseModel)` | **Primary hook** — full response body received. Default implementation is empty |
| `OnReceivedResponseError(ResponseModel, string)` | Header or body parse error — the client disconnects immediately after |

> **Not verified in this file:** whether sends are held/rejected until the TLS handshake completes, or whether there's an `OnConnectedOrHandshaked`-style hook. That would live in `SslClient`, not here — check that source before documenting it.

---

## Receive / Disconnect Behavior

- `OnReceived` parses incrementally: header first (fires `OnReceivedResponseHeader`), then body (fires `OnReceivedResponse` and clears `Response` for reuse). A parse error at either stage fires `OnReceivedResponseError`, clears `Response`, and disconnects.
- `OnDisconnected` flushes a response whose body was still pending (calls `OnReceivedResponse` with the partial response), then disposes and releases both `Request` and `Response` back to the pool before calling `base.OnDisconnected()`.

---

## Usage

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

TLS state and connect/disconnect are inherited from `SslClient`. All base client methods are inherited from `ClientTransport`.

---