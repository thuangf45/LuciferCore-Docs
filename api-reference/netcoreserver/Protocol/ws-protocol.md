# WebSocket Protocol Layer

**Namespace:** `LuciferCore.NetCoreServer.Protocol.Ws`

The WebSocket protocol layer implements RFC 6455 frame encoding/decoding, the HTTP upgrade handshake, and all send/receive operations. All types in this layer are `internal` — they are not part of the public API. This page documents the architecture for contributors and developers who need to understand how WebSocket framing, masking, and multicast work under the hood.

---

## Overview

```
WsSession / WssSession / WsClient / WssClient
        ↓ delegates to
┌──────────────────────────────────────┐
│  WsSessionProtocol                   │  ← per-session: frame send/receive
│  WsClientProtocol  (extends Session) │  ← client-side: connect + close
│  WsServerProtocol                    │  ← server-side: multicast + CloseAll
└──────────────────────────────────────┘
        ↓ uses
WebSocket (core framing engine)
        ↓ routes events via
IWebSocket (internal interface)
        ↓ dispatches to
OnWsReceived() → Lucifer.Dispatch() → [Handler] methods
```

---

## IWebSocket

Internal interface implemented by WebSocket session and client classes to receive parsed protocol events.

```csharp
internal interface IWebSocket
```

| Method | Called when |
|---|---|
| `OnWsConnecting(RequestModel)` | Client-side: before sending upgrade request. Can modify request headers |
| `OnWsConnected(ResponseModel)` | Client-side: upgrade response received and validated |
| `OnWsConnecting(RequestModel, ResponseModel)` | Server-side: validate the upgrade request. Return `false` to reject |
| `OnWsConnected(RequestModel)` | Server-side: upgrade complete, connection ready |
| `OnWsDisconnecting()` | WebSocket connection is about to close |
| `OnWsDisconnected()` | WebSocket connection closed |
| `OnWsReceived(byte[] buffer, long offset, long size)` | Binary or text data frame received — primary receive hook |
| `OnWsClose(byte[] buffer, long offset, long size, int status)` | Close frame received. Default status is `1000` (Normal Closure) |
| `OnWsPing(byte[] buffer, long offset, long size)` | Ping frame received |
| `OnWsPong(byte[] buffer, long offset, long size)` | Pong frame received |
| `OnWsError(string error)` | Protocol-level error |
| `OnWsError(SocketError error)` | Socket-level error |
| `SendUpgrade(ResponseModel response)` | Server-side: sends the HTTP 101 upgrade response |

---

## WebSocket (Core Framing Engine)

The `WebSocket` class is the RFC 6455 implementation. It handles the HTTP upgrade handshake and all frame encoding/decoding.

### Opcode Constants

| Constant | Value | Description |
|---|---|---|
| `WS_FIN` | `0x80` | FIN bit — marks the final fragment of a message |
| `WS_TEXT` | `0x01` | Text frame opcode |
| `WS_BINARY` | `0x02` | Binary frame opcode |
| `WS_CLOSE` | `0x08` | Close frame opcode |
| `WS_PING` | `0x09` | Ping frame opcode |
| `WS_PONG` | `0x0A` | Pong frame opcode |

Frames are built by ORing `WS_FIN` with the opcode: `WS_FIN | WS_BINARY`, `WS_FIN | WS_TEXT`, etc.

### Handshake

```
Server-side: PerformServerUpgrade(RequestModel request)
    → validates Upgrade: websocket header
    → reads Sec-WebSocket-Key
    → computes Sec-WebSocket-Accept (SHA-1 + Base64)
    → calls IWebSocket.OnWsConnecting(request, response) for validation
    → calls IWebSocket.SendUpgrade(response) → sends HTTP 101
    → _wsHandshaked = true

Client-side: PerformClientUpgrade(ResponseModel response, long id)
    → validates HTTP 101 status
    → verifies Sec-WebSocket-Accept against sent nonce
    → _wsHandshaked = true
    → calls IWebSocket.OnWsConnected(response)
```

### Frame Building

```csharp
int BuildFrame(byte opcode, bool masking, ReadOnlySpan<byte> payload, Span<byte> output, int status = 0)
static int ComputeFrameSize(int payloadLength, bool isCloseFrame, bool mask)
```

`BuildFrame` writes a complete RFC 6455 frame into `output`. `ComputeFrameSize` pre-computes the exact byte count so the output buffer can be sized precisely before building — no reallocation needed.

**Masking:** client→server frames are always masked (per RFC 6455). Server→client frames are never masked. `WsSessionProtocol` sets `_masking = true` when the session is a `ClientTransport`.

### Frame Reception

```csharp
void PrepareReceiveFrame(ReadOnlySpan<byte> buffer)
long RequiredReceiveFrameSize()
void ClearWsBuffers()
```

`PrepareReceiveFrame` is the incremental frame parser — call it repeatedly with incoming bytes. It accumulates fragments across multiple `WS_BINARY`/`WS_TEXT` frames until the FIN frame arrives, then triggers the appropriate `IWebSocket` callback.

`RequiredReceiveFrameSize()` returns how many bytes are needed to complete the current frame header or payload — used by synchronous `ReceiveBinary` / `ReceiveText` in `WsSessionProtocol`.

---

## WsSessionProtocol

Per-session WebSocket protocol handler. Wraps a `WebSocket` instance and exposes a clean send API that handles frame building, pooled buffer management, and masking transparently.

**Send API (all accept `ReadOnlySpan<byte>` and `ReadOnlySpan<char>`):**

| Method group | Frame opcode |
|---|---|
| `SendText` / `SendTextAsync` | `WS_FIN \| WS_TEXT` |
| `SendBinary` / `SendBinaryAsync` | `WS_FIN \| WS_BINARY` |
| `SendClose` / `SendCloseAsync` | `WS_FIN \| WS_CLOSE` |
| `SendPing` / `SendPingAsync` | `WS_FIN \| WS_PING` |
| `SendPong` / `SendPongAsync` | `WS_FIN \| WS_PONG` |
| `Close(status, payload)` | Close frame + `Disconnect()` |

**Frame send path:**

```
SendBinaryAsync(data)
    → SendAsyncFrame(WS_FIN | WS_BINARY, data)
        → BuildFrame(opcode, data)
            → ComputeFrameSize()   ← exact size, no realloc
            → Lucifer.Rent<Buffer>()
            → WebSocket.BuildFrame() → writes frame into buffer
        → Session.SendAsync(buffer)  ← ownership transfer, pooled
```

**Synchronous receive:**

`ReceiveBinary()` and `ReceiveText()` implement a synchronous receive loop using `RequiredReceiveFrameSize()` to pull exactly the right number of bytes from the socket per iteration, reassembling fragmented messages without intermediate allocation.

**Masking:** `_masking = true` when the underlying session is a `ClientTransport` — client frames are masked per RFC 6455.

---

## WsClientProtocol

Extends `WsSessionProtocol` with client-specific connect and async close methods. Tracks `_syncConnect` to distinguish sync vs async connection path (needed for the handshake flow in the client session).

---

## WsServerProtocol

Server-side WebSocket protocol. Handles multicast and `CloseAll` by building a single frame and multicasting it to all sessions via `ServerTransport.Multicast(Buffer)` — one allocation for all connected clients.

**Multicast API:**

| Method | Frame opcode |
|---|---|
| `MulticastText` | `WS_FIN \| WS_TEXT` |
| `MulticastBinary` | `WS_FIN \| WS_BINARY` |
| `MulticastPing` | `WS_FIN \| WS_PING` |
| `CloseAll(status, payload)` | `WS_FIN \| WS_CLOSE` + `DisconnectAll()` |

**Multicast frame path:**

```
MulticastBinary(data)
    → MulticastFrame(WS_FIN | WS_BINARY, data)
        → ComputeFrameSize()        ← masking = false (server→client)
        → Lucifer.Rent<Buffer>()
        → WebSocket.BuildFrame()    → writes frame
        → ServerTransport.Multicast(buffer)  ← one buffer, all sessions
```

---

## Remarks

- `WebSocket` is instantiated once per session/client via `WsSessionProtocol` — frame state (`_wsHandshaked`, `_wsFrameReceived`, `_wsFinalReceived`) is per-instance and not shared.
- Receive buffers (`_wsReceiveFrameBuffer`, `_wsReceiveFinalBuffer`) are pre-allocated fields on the `WebSocket` instance — no per-frame allocation for the receive path.
- The `_wsReceiveLock` and `_wsSendLock` objects guard concurrent access to receive and send buffers respectively.
- Close frames include a 2-byte status code prepended to the payload (per RFC 6455 section 5.5.1) — handled automatically by `BuildFrame` when `storeStatus` is true.
