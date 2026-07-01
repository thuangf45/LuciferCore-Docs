# ClientTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

`ClientTransport` is the base class for client-side transport connections.

It extends `SessionTransport` and adds:
- connect/reconnect flow
- client socket setup
- endpoint identity handling

```csharp
public class ClientTransport : SessionTransport
```

---

## Construction

Constructors are `internal` (typically exposed via derived/factory client type):

```csharp
internal ClientTransport(IPAddress address, int port)
internal ClientTransport(string address, int port)
internal ClientTransport(IPEndPoint endpoint)
internal ClientTransport(DnsEndPoint endpoint)
internal ClientTransport(EndPoint endpoint, string address, int port)
```

For `DnsEndPoint`:
- `Address` keeps original hostname
- `Endpoint` resolves to concrete `IPEndPoint` (IPv4 preferred)

---

## Identity properties

| Property | Type | Meaning |
|---|---|---|
| `Address` | `string` | target server address/hostname |
| `Port` | `int` | target port |
| `Endpoint` | `EndPoint` | resolved endpoint used for connect |
| `IsConnecting` | `bool` | async connect in progress flag |

---

## Connect / reconnect API

```csharp
bool Connect()
bool Disconnect()
bool Reconnect()

bool ConnectAsync()
bool DisconnectAsync()
bool ReconnectAsync()
```

Notes:
- endpoint is fixed by constructor (no public `Connect(address, port)` overload)
- `DisconnectAsync()` currently delegates to sync disconnect
- `ReconnectAsync()` disconnects, waits until disconnected, then starts async connect

---

## Connect flow (sync)

`Connect()` does (high-level):

1. rent/setup send/receive buffers
2. create socket (`CreateSocket`)
3. client socket setup (`ClientSetUp`)
4. fire `OnConnecting`
5. execute connect (`TryConnect`)
6. apply socket options
7. reserve buffer capacity
8. fire `OnConnected`
9. start receive
10. run handshake logic

If any step fails, returns `false`.

---

## Socket options source

Client socket options come from `SessionInfo.Options`:

| Option | Meaning |
|---|---|
| `OptionDualMode` | IPv4+IPv6 dual mode (IPv6 socket only) |
| `OptionKeepAlive` | TCP keepalive |
| `OptionTcpKeepAliveTime` | keepalive idle threshold |
| `OptionTcpKeepAliveInterval` | keepalive interval |
| `OptionTcpKeepAliveRetryCount` | keepalive retry count |
| `OptionNoDelay` | disable Nagle |

---

## Overridable members

```csharp
protected virtual Socket CreateSocket()
protected virtual void ClientSetUp()
protected virtual bool TryConnect()
protected override void ApplySocketOptions()
internal override void ProcessConnect(SocketAsyncEventArgs e)
```

Use these to customize connect behavior (e.g. TLS clients).

---

## Inherited behavior

All send/receive/disconnect hooks are inherited from `SessionTransport`, including:
- lifecycle callbacks
- async send/receive pipeline
- socket error handling