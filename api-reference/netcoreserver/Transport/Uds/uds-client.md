# UdsClient

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDS`

`UdsClient` is the Unix Domain Socket client class.  
It extends `StreamClientTransport`.

```csharp
public class UdsClient : StreamClientTransport
```

> UDS is supported on Linux and macOS.  
> On Windows, it requires Windows 10 build 17063 or later.

---

## Constructors

```csharp
public UdsClient(string path)                        // example: "/tmp/myapp.sock"
public UdsClient(UnixDomainSocketEndPoint endpoint)
```

Use a filesystem socket path instead of IP/port.

---

## Socket behavior

`UdsClient` uses Unix stream sockets (`AddressFamily.Unix`).

- `ClientSetUp()` is empty
- `ApplySocketOptions()` is empty

TCP-specific options are not used for UDS.

---

## Quick usage

```csharp
var client = new UdsClient("/tmp/myapp.sock");
client.ConnectAsync();
client.SendAsync("ping"u8);
```

---

## Custom client example

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

`UdsClient` adds no new public connect/send/receive methods.

Use inherited APIs from `StreamClientTransport` / `ClientTransport` / `SessionTransport`:

- connect/reconnect/disconnect
- `Send<T>(...)`, `SendAsync<T>(...)`
- receive methods/hooks
- lifecycle hooks/events
- `Dispose()`

---

## Notes

- Use `UdsClient` for local IPC over Unix domain sockets.
- Put app/protocol logic in a derived client class.