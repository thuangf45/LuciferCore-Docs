# UdpClient

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDP`

`UdpClient` is the UDP client class.  
It extends `ClientTransport` and communicates using UDP datagrams (`SendTo` / `ReceiveFrom`).

```csharp
public class UdpClient : ClientTransport
```

---

## Constructors

```csharp
public UdpClient(IPAddress address, int port)
public UdpClient(string address, int port)
public UdpClient(DnsEndPoint endpoint)
public UdpClient(IPEndPoint endpoint)
public UdpClient(EndPoint endpoint, string address, int port)
```

Use the constructor that matches your target server address input.

---

## Public API (inherited + UDP behavior)

From `ClientTransport` / `SessionTransport`:

- `Connect()`, `ConnectAsync()`
- `Disconnect()`, `DisconnectAsync()`
- `Reconnect()`, `ReconnectAsync()`
- `Send<T>(...)`, `SendAsync<T>(...)`
- receive hooks and lifecycle hooks
- metrics/options access
- `Dispose()`

`UdpClient` keeps the same public usage model while implementing UDP-specific socket operations internally.

---

## Protected hooks you can override

| Method | Description |
|---|---|
| `CreateSocket()` | Customize UDP socket creation |
| `ClientSetUp()` | Apply client socket options before use |
| `TryConnect()` | Customize UDP bind/connect preparation |
| `OnReceived(EndPoint endpoint, byte[] buffer, long offset, long size)` | Datagram received from remote endpoint |
| `OnSent(EndPoint endpoint, long sent)` | Datagram sent to remote endpoint |

You can also override inherited hooks such as:

- `OnConnecting()`, `OnConnected()`
- `OnDisconnecting()`, `OnDisconnected()`
- `OnReceived(byte[] buffer, long offset, long size)`
- `OnSent(long sent, long pending)`
- `OnEmpty()`

---

## Quick usage

```csharp
var client = new UdpClient("127.0.0.1", 8080);
client.Connect();
client.Send("hello"u8);
```

---

## Example custom client

```csharp
public class TestUdpClient : UdpClient
{
    public TestUdpClient(string address, int port) : base(address, port) { }

    protected override void OnReceived(byte[] buffer, long offset, long size)
    {
        var text = System.Text.Encoding.UTF8.GetString(buffer, (int)offset, (int)size);
        Console.WriteLine($"Client Received: {text}");
    }
}
```

---

## Notes

- `UdpClient` is datagram-oriented (message boundaries are preserved by UDP datagrams).
- Remote endpoint filtering can be handled in `OnReceived(EndPoint, ...)` when needed.
```