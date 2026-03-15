# HttpsClient

**Namespace:** `LuciferCore.NetCoreServer.Client`

Secure HTTPS client. Extends `SslClient` with HTTP request building, response parsing, and a `Task`-based async request API with configurable timeout over TLS.

```csharp
public class HttpsClient : SslClient, IHttp
```

---

## Constructors

```csharp
public HttpsClient(SslContext context, IPAddress address, int port)
public HttpsClient(SslContext context, string address, int port)
public HttpsClient(SslContext context, IPEndPoint endpoint)
public HttpsClient(SslContext context, DnsEndPoint endpoint)
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
| `OnConnectedOrHandshaked()` | After **TLS handshake** completes — pending request is sent here |
| `OnReceivedResponseHeader(ResponseModel)` | Response headers parsed |
| `OnReceivedResponse(ResponseModel)` | **Primary hook** — full response received |
| `OnReceivedResponseError(ResponseModel, string)` | Response parse error |

> For `HttpsClient`, `OnConnectedOrHandshaked` fires after the TLS handshake (`OnHandshaked`), not just TCP connect.

---

## Usage

```csharp
var context = new SslContext(SslProtocols.Tls12);
var client = new HttpsClient(context, "api.example.com", 443);

var response = await client.SendGetRequest("/api/users");

// Skip cert validation in dev
var devContext = new SslContext(SslProtocols.Tls12, (_, _, _, _) => true);
var devClient = new HttpsClient(devContext, "localhost", 8443);
```

---

## Inherited API

TLS state and connect/disconnect are inherited from `SslClient`. See [SslClient](ssl-client.md). All base client methods are inherited from `ClientTransport`. See [ClientTransport](transport-client.md).
