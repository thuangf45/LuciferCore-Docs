# StreamClientTransport

**Namespace:** `LuciferCore.NetCoreServer.Transport.Core`

Concrete stream client. Connects to a remote server using batched async send and `SocketAsyncEventArgs`-driven receive. Supports any `EndPoint` type — `IPEndPoint`, `DnsEndPoint`, or custom endpoints for other transport families (UDS, etc.).

```csharp
public class StreamClientTransport : ClientTransport
```

---

## Constructors

```csharp
public StreamClientTransport(IPAddress address, int port)
public StreamClientTransport(string address, int port)
public StreamClientTransport(IPEndPoint endpoint)
public StreamClientTransport(DnsEndPoint endpoint)
```

---

## Connect Pipeline

Async connection uses a dedicated `SocketAsyncEventArgs` for the connect operation:

```
ConnectAsync()
    → HandleAsyncConnect() — sets up _connectEventArg
    → Socket.ConnectAsync(_connectEventArg)
    → ProcessConnect(e)    — on completion
    → OnConnected()        — your hook
    → FirstTryReceive()    — starts receive loop
```

---

## Send Pipeline

Same batching strategy as `StreamSessionTransport`:

```
SendAsync(buffer)
    → HandleAsyncSend()  — adds ArraySegment to _batchList
    → HandleAsyncFlush() — Socket.SendAsync(_batchList) in one syscall
    → _batchList.Clear()
```

---

## Receive Pipeline

`SocketAsyncEventArgs`-driven loop with synchronous fast-path — identical to `StreamSessionTransport`.

---

## Disconnect

On disconnect or connection loss:

- `CancelAsyncOperations()` cancels the in-progress connect `SocketAsyncEventArgs` if still connecting
- Event handlers are unsubscribed via `CancelAsyncCompleted()`
- Socket is shut down with `Socket.Shutdown(SocketShutdown.Both)`

---

## Usage

```csharp
var client = new StreamClientTransport("127.0.0.1", 8080);
client.ConnectAsync();
client.SendAsync("hello"u8);
client.Disconnect();
```

Extend to override lifecycle hooks:

```csharp
public class ChatClient : StreamClientTransport
{
    public ChatClient(string host, int port) : base(host, port) { }

    protected override void OnConnected()
        => Console.WriteLine("Connected");

    protected override void OnReceived(byte[] buffer, long offset, long size)
        => Console.WriteLine(Encoding.UTF8.GetString(buffer, (int)offset, (int)size));

    protected override void OnDisconnected()
        => Console.WriteLine("Disconnected");
}
```

---

## Inherited API

All send, receive, connect/disconnect, and lifecycle hook methods are inherited from `ClientTransport` and `SessionTransport`. See [ClientTransport](transport-client.md) and [SessionTransport](transport-session.md).
