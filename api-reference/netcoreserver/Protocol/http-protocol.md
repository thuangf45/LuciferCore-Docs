# HTTP Protocol Layer

**Namespace:** `LuciferCore.NetCoreServer.Protocol.Http`

The HTTP protocol layer sits between the raw socket transport and the application-level session/client classes. All types in this layer are `internal` — they are not part of the public API. This page documents the architecture for contributors and for developers who need to understand how HTTP parsing, static content, and request/response lifecycle work under the hood.

---

## Overview

```
HttpSession / HttpsSession / HttpClient / HttpsClient
        ↓ delegates to
┌─────────────────────────────────┐
│  HttpSessionProtocol            │  ← server-side: parses requests, serves cache
│  HttpClientProtocol             │  ← client-side: sends requests, parses responses
│  HttpServerProtocol             │  ← server-side: manages FileCache / static content
└─────────────────────────────────┘
        ↓ routes events via
IHttp (internal interface)
        ↓ dispatches to
OnReceivedRequest() → Lucifer.Dispatch() → [Handler] methods
```

---

## IHttp

Internal interface implemented by HTTP session and client classes to receive parsed protocol events.

```csharp
internal interface IHttp
```

| Method | Called when |
|---|---|
| `OnReceivedRequest(RequestModel)` | A complete HTTP request is ready for dispatch |
| `OnReceivedRequestHeader(RequestModel)` | Request headers have been parsed (body may still be pending) |
| `OnReceivedRequestInternal(RequestModel)` | Pre-dispatch hook — rate limiting and cache lookup happen here |
| `OnReceivedCachedRequest(RequestModel, byte[])` | A matching static file is found in the cache — sends directly without dispatch |
| `OnReceivedRequestError(RequestModel, string)` | Parse error encountered |
| `OnReceivedResponse(ResponseModel)` | A complete HTTP response is received (client-side) |
| `OnReceivedResponseHeader(ResponseModel)` | Response headers parsed (client-side) |
| `OnReceivedResponseError(ResponseModel, string)` | Response parse error (client-side) |
| `GetStaticPath(RequestModel)` | Returns the static file path to serve for this request |

---

## HttpSessionProtocol

Internal class that handles the incremental HTTP request parsing loop for a single server-side session.

**Responsibilities:**

- Incrementally parses raw bytes into a `RequestModel` via `ReceiveHeader` + `ReceiveBody`.
- Enforces a **65 536-byte body limit** — requests exceeding this while `Lucifer.Overloaded` is `false` are dropped and the session is disconnected.
- On parse error, fires `OnReceivedRequestError` and disconnects.
- For `GET` requests, performs a `FileCache` lookup before dispatching. If a cached response exists, it is sent directly without going through the handler pipeline.
- On session disconnect, flushes any pending partial body via `OnReceivedRequestInternal`.

**Static path resolution** (`GetStaticPath`):

```
Request URL
    → strip query string (?...)
    → check Mapping dict       → return mapped file
    → ends with ".html"?       → return /404
    → exists in FileCache?     → return path
    → fallback                 → return /404
```

---

## HttpServerProtocol

Internal class that owns the server's `FileCache` and manages static content.

**Responsibilities:**

- Wraps `FileCache` with a pre-configured `InsertHandler` that builds a full HTTP `200 OK` response (including `Content-Type` from MIME table and `Cache-Control` header) for each file before inserting it into the cache.
- `AddStaticContent()` calls `Cache.EnsureSystemFiles()` first, then `Cache.InsertPath()` — so framework default files are always present regardless of the directory contents.
- Default `timeout` is **1 hour** — files are re-read and re-inserted after expiry.

**Cache entry format:** each cached entry is a pre-built raw HTTP response (`ResponseModel.Cache.Data`), ready to send over the wire without any runtime allocation.

---

## HttpClientProtocol

Internal class that manages the HTTP request/response cycle for a client connection, including `Task`-based async request support.

**Responsibilities:**

- Holds a `TaskCompletionSource<ResponseModel>` (`_tcs`) that is resolved when a complete response is received.
- `SendRequest(RequestModel, TimeSpan?)` — the primary async API:
  1. Validates the request.
  2. Creates `_tcs`.
  3. Connects if not yet connected, or sends immediately if already connected.
  4. Wraps everything in `Lucifer.Deadline<T>` — on timeout, disconnects and returns an empty `ResponseModel`.
- `OnConnectedOrHandshaked()` — called after TLS handshake completes. Sends the pending request if `_tcs` is set.
- `OnReceived()` — incrementally parses the response via `ReceiveHeader` + `ReceiveBody`. Resolves `_tcs` when the body is complete.
- `OnDisconnected()` — resolves `_tcs` with a partial response if the body was still pending (connection closed before body complete).

**Assertion:** only one in-flight request per client is supported — concurrent `SendRequest` calls before awaiting are caught with `Debug.Assert`.

---

## Remarks

- These classes are instantiated by `HttpSession`, `HttpsSession`, `HttpClient`, and `HttpsClient` — never directly.
- The `FileCache` in `HttpServerProtocol` is shared across all sessions of the same server instance (passed into each `HttpSessionProtocol` at construction).
- Static content is served without going through `Lucifer.Dispatch()` — the cached raw response bytes are sent directly over the socket. This is the zero-allocation static file path.
- The 65 536-byte body limit in `HttpSessionProtocol` is a server-side protection against large payload attacks. It is only enforced when `Lucifer.Overloaded` is `false`.
