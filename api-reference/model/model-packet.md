# PacketModel

**Namespace:** `LuciferCore.Model`

`PacketModel` is the zero-copy payload container for all incoming WebSocket binary frames. It wraps a pooled `Buffer` and exposes the URL and body as zero-allocation `Span<byte>` views into the underlying memory — no copies, no intermediate strings.

It extends `PooledObject` and implements `IDisposable`. Always obtain instances via `Lucifer.Rent<PacketModel>()` and release them with `using var _ = data;` or `Lucifer.Return(model)`.

---

## Packet Wire Format

Every incoming WebSocket binary frame must conform to this binary layout:

```
┌──────────────┬──────────────┬──────────────────┬────────────────────┐
│  MAGIC (4B)  │ UrlLen (4B)  │   URL (UrlLen B) │   Body (remaining) │
│  0x4643554C  │  little-end  │   UTF-8 string   │   arbitrary bytes  │
└──────────────┴──────────────┴──────────────────┴────────────────────┘
 offset 0       offset 4       offset 8            offset 8 + UrlLen
```

| Field | Size | Description |
|---|---|---|
| `MAGIC` | 4 bytes | Constant `0x4643554C` (little-endian). Validates packet integrity |
| `UrlLen` | 4 bytes | Length of the URL field in bytes (little-endian `int32`) |
| `URL` | `UrlLen` bytes | UTF-8 route path used for dispatch (e.g. `/v1/wss/ChatMessage`) |
| `Body` | remaining bytes | Arbitrary payload — JSON, binary, or any format your handler expects |

---

## Constants

```csharp
public const int MAGIC           = 0x4643554C;
public const int MagicLength     = 4;
public const int UrlLengthFieldSize = 4;
public const int UrlOffset       = 8; // MagicLength + UrlLengthFieldSize
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `Buffer` | `Buffer?` | The attached pooled buffer holding the raw packet bytes. Both getter and setter call `CheckSafety()` — accessing `Buffer` on a disposed `PacketModel` throws `ObjectDisposedException` |
| `IsValid` | `bool` | `true` if the buffer has a valid MAGIC header and a URL that fits within the buffer bounds |
| `UrlView` | `ByteString` | Zero-copy UTF-8 view of the URL field. Falls back to `UrlDefault` if the packet is invalid |
| `Body` | `ReadOnlySpan<byte>` | Zero-copy span of the body bytes. If the packet is invalid, returns the entire buffer as fallback |
| `UrlDefault` | `ByteString` (static) | Default URL used when a packet is invalid. Defaults to `/v1/wss/Default`. Configurable |

---

## Methods

### `BodyAs<T>()`

Deserializes the body as a typed object. The result is cached — subsequent calls with the same type return the cached instance without re-deserializing.

```csharp
public T? BodyAs<T>() where T : class
```

```csharp
var message = data.BodyAs<ChatMessageDto>();
```

> Deserialization is performed directly from the body span with no intermediate byte array allocation.

---

### `ToBufferWithMagic<TBuffer, TUrl>()`

Builds an outbound packet buffer conforming to the wire format. Used internally to construct error response frames.

```csharp
public Buffer ToBufferWithMagic<TBuffer, TUrl>(
    ReadOnlySpan<TBuffer> buffer,
    ReadOnlySpan<TUrl> url)
    where TBuffer : unmanaged
    where TUrl : unmanaged
```

The returned `Buffer` is rented from the pool. The caller is responsible for returning it.

---

### `Attach(Buffer newBuffer)`

Attaches a new buffer to the packet, returning the previous buffer to the pool if present. Clears the body deserialization cache.

```csharp
public void Attach(Buffer newBuffer)
```

---

### `GetBytes(object data)` *(static)*

Serializes an object to a UTF-8 JSON byte array. Returns the array directly if `data` is already `byte[]`, or an empty array if `data` is `null`.

```csharp
public static byte[] GetBytes(object data)
```

---

## Lifecycle & Pooling

`PacketModel` is a pooled object. The internal `Reset()` method returns the attached `Buffer` to the pool and clears the deserialization cache. This is called automatically when the instance is returned to the pool.

```csharp
// Correct usage inside a handler — return to pool after reading
public void SendChat([Session] WssSession session, [Data] PacketModel data)
{
    using var _ = data;                              // returns to pool on scope exit
    ((WssServer)session.Server).MulticastBinary(data.Buffer);
}
```

> Always wrap `data` in a `using` statement or call `Lucifer.Return(data)` explicitly. Failing to do so causes the buffer to remain pinned in memory and the pool slot to be lost.

---

## Remarks

- `UrlView` returns a `ByteString` that is a **view** into the buffer — it holds no independent allocation. Do not use it after `Dispose()` is called.
- `Body` is likewise a zero-copy span. Copy the bytes if you need them to outlive the handler scope.
- `IsValid` performs a zero-copy MAGIC check and bounds check on every call. It is inlined and costs two `BinaryPrimitives.ReadInt32LittleEndian` calls.
- `UrlDefault` is a static field and can be changed at startup to customize the fallback route for invalid frames.
