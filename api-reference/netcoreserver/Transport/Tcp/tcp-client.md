# TcpClient

**Namespace:** `LuciferCore.NetCoreServer.Transport.TCP`

`TcpClient` is the default TCP client class.  
It extends `StreamClientTransport`.

```csharp
public class TcpClient : StreamClientTransport
```

---

## Constructors

```csharp
public TcpClient(string host)                          // uses port 80
public TcpClient(DnsEndPoint endpoint)
public TcpClient(IPAddress address, int port)
public TcpClient(string address, int port)
public TcpClient(IPEndPoint endpoint)
public TcpClient(EndPoint endpoint, string address, int port)
```

Choose the constructor that matches your input (host, IP, or endpoint).

---

## Quick usage

```csharp
var client = new TcpClient("127.0.0.1", 9000);
client.ConnectAsync();
client.SendAsync("ping"u8);
```

---

## Custom client example

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

`TcpClient` adds no new public connect/send/receive methods.

Use inherited APIs from `StreamClientTransport` / `ClientTransport` / `SessionTransport`:

- connect/reconnect/disconnect
- `Send<T>(...)`, `SendAsync<T>(...)`
- receive methods/hooks
- lifecycle hooks/events
- `Dispose()`

---

## Notes

- Use `TcpClient` as the default TCP client.
- Put app/protocol logic in a derived client class.