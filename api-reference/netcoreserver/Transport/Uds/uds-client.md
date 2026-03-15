# UdsClient

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDS`

Unix Domain Socket (UDS) client. Extends `StreamClientTransport` using a filesystem socket path instead of IP:port. TCP socket options and keep-alive are not applied.

```csharp
public class UdsClient : StreamClientTransport
```

> UDS is only available on Linux and macOS. On Windows, UDS support requires Windows 10 build 17063 or later.

---

## Constructors

```csharp
public UdsClient(string path)                        // e.g. "/tmp/myapp.sock"
public UdsClient(UnixDomainSocketEndPoint endpoint)
```

---

## Socket

Uses `SocketType.Stream` with `ProtocolType.Unspecified` and `AddressFamily.Unix`. `ClientSetUp()` and `ApplySocketOptions()` are both no-ops — no TCP-specific configuration is applied.

---

## Usage

```csharp
var client = new UdsClient("/tmp/myapp.sock");
client.ConnectAsync();
client.SendAsync("ping"u8);
```

Extend to override lifecycle hooks:

```csharp
public class MyClient : UdsClient
{
    public MyClient(string path) : base(path) { }

    protected override void OnConnected()
        => SendAsync("hello"u8);

    protected override void OnReceived(byte[] buffer, long offset, long size)
        => Console.WriteLine(Encoding.UTF8.GetString(buffer, (int)offset, (int)size));
}
```

---

## Inherited API

All connect/disconnect, send/receive, statistics, and lifecycle hook methods are inherited from `StreamClientTransport` and `ClientTransport`. See [StreamClientTransport](transport-stream-client.md) and [ClientTransport](transport-client.md).
