# TcpClient

**Namespace:** `LuciferCore.NetCoreServer.Transport.TCP`

Concrete TCP client. Extends `StreamClientTransport` — inherits batched async send, `SocketAsyncEventArgs`-driven receive, and all `ClientTransport` connect/reconnect logic.

```csharp
public class TcpClient : StreamClientTransport
```

---

## Constructors

```csharp
public TcpClient(IPAddress address, int port)
public TcpClient(string address, int port)
public TcpClient(IPEndPoint endpoint)
public TcpClient(DnsEndPoint endpoint)
```

---

## Usage

```csharp
var client = new TcpClient("127.0.0.1", 9000);
client.ConnectAsync();
client.SendAsync("ping"u8);
```

Extend to override lifecycle hooks:

```csharp
public class MyClient : TcpClient
{
    public MyClient(string host, int port) : base(host, port) { }

    protected override void OnConnected()
        => SendAsync("hello"u8);

    protected override void OnReceived(byte[] buffer, long offset, long size)
        => Console.WriteLine(Encoding.UTF8.GetString(buffer, (int)offset, (int)size));
}
```

---

## Inherited API

All connect/disconnect, send/receive, socket options, statistics, and lifecycle hook methods are inherited from `StreamClientTransport`, `ClientTransport`, and `SessionTransport`. See [StreamClientTransport](transport-stream-client.md) and [ClientTransport](transport-client.md).
