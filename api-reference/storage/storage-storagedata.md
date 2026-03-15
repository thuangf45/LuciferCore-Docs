# StorageData

**Namespace:** `LuciferCore.Storage`

`StorageData` is the central UTF-8 constant catalog for LuciferCore's HTTP and WebSocket processing pipeline. All values are pre-encoded as `ByteString` (zero-allocation UTF-8 byte strings) at static initialization time and reused across every request and response without any runtime allocation.

You will not typically reference `StorageData` directly — it is consumed internally by `RequestModel`, `ResponseModel`, and the handler pipeline. It is documented here for completeness and for advanced use cases such as custom response building.

---

## Constant Groups

### Protocol Tokens

| Field | Value |
|---|---|
| `Http11Protocol` | `HTTP/1.1` |
| `Http2Protocol` | `HTTP/2` |
| `Http3Protocol` | `HTTP/3` |
| `UnknownStatus` | `Unknown Status` |

### HTTP Methods

Standard methods: `GetMethod`, `PostMethod`, `PutMethod`, `DeleteMethod`, `HeadMethod`, `OptionsMethod`, `TraceMethod`, `ConnectMethod`, `PatchMethod`.

WebDAV extensions: `PropFindMethod`, `PropPatchMethod`, `MkColMethod`, `CopyMethod`, `MoveMethod`, `LockMethod`, `UnlockMethod`, `ReportMethod`, `MkActivityMethod`, `CheckoutMethod`, `MergeMethod`, `MSearchMethod`, `NotifyMethod`, `SubscribeMethod`, `UnsubscribeMethod`.

### Content Headers

| Field | Header name |
|---|---|
| `ContentType` | `Content-Type` |
| `ContentLength` | `Content-Length` |
| `ContentEncoding` | `Content-Encoding` |
| `ContentDisposition` | `Content-Disposition` |
| `ContentLanguage` | `Content-Language` |
| `ContentLocation` | `Content-Location` |
| `ContentRange` | `Content-Range` |
| `ContentSecurityPolicy` | `Content-Security-Policy` |

### Cookie Headers

| Field | Value |
|---|---|
| `Cookie` | `Cookie` |
| `SetCookie` | `Set-Cookie` |
| `MaxAge` | `Max-Age` |
| `Domain` | `Domain` |
| `Path` | `Path` |
| `Secure` | `Secure` |
| `HttpOnly` | `HttpOnly` |
| `SameSite` | `SameSite` |
| `SameSiteStrict` | `SameSite=Strict` |
| `Expires` | `Expires` |

### CORS Headers

| Field | Header / Value |
|---|---|
| `ACAM` | `Access-Control-Allow-Methods` |
| `ACAMethods` | `GET, POST, PUT, DELETE, OPTIONS` |
| `ACAH` | `Access-Control-Allow-Headers` |
| `ACAHeaders` | `X_Token_Authorization` |
| `ACAC` | `Access-Control-Allow-Credentials` |
| `ACACValue` | `true` |
| `AllowHeader` | `Allow` |
| `AllowOptions` | `HEAD,GET,POST,PUT,DELETE,OPTIONS,TRACE` |

### WebSocket Handshake Constants

`WebSocketProtocol`, `KeepAliveUpgrade`, `UpgradeHeader`, `ConnectionHeader`, `SecWebSocketKeyHeader`, `SecWebSocketVersionHeader`, `SecWebSocketAcceptHeader`, `SecWebSocketProtocolHeader`, `SecWebSocketExtensionsHeader`, `SecWebSocketKeyGuid`, `WebSocketVersion` (`"13"`).

### Common Delimiters

| Field | Value |
|---|---|
| `CrLf` | `\r\n` |
| `CrLfCrLf` | `\r\n\r\n` |
| `ColonSpace` | `: ` |
| `SemicolonSpace` | `; ` |
| `Space` | ` ` |
| `Equal` | `=` |
| `Slash` | `/` |
| `Dot` | `.` |

### Lookup Tables

| Field | Type | Description |
|---|---|---|
| `MimeTable` | `Utf8Map<ByteString>` | File extension → MIME type. Case-insensitive. Used by `ResponseModel.SetContentType()` |
| `StatusPhrases` | `Dictionary<int, ByteString>` | HTTP status code → reason phrase. Used by `ResponseModel.SetBegin()` |

`MimeTable` covers all common web extensions: `.html`, `.css`, `.js`, `.json`, `.png`, `.jpg`, `.svg`, `.wasm`, `.woff2`, and more.

`StatusPhrases` covers the full standard HTTP status code range (1xx–5xx).

---

## Remarks

- All fields are `public static readonly ByteString` — they are initialized once at class load and never mutated.
- Accessing any field is allocation-free — `ByteString` is a value-type wrapper over a pre-allocated byte array.
- The `MimeTable` is constructed with `ignoreCase: true`, so `.HTML` and `.html` resolve identically.
- Do not add entries to `MimeTable` or `StatusPhrases` at runtime — they are shared global state and are not thread-safe for writes after initialization.
