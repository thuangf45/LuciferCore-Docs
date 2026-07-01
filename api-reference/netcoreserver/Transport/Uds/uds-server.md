# UdsServer

**Namespace:** `LuciferCore.NetCoreServer.Transport.UDS`

Unix Domain Socket (UDS) server. Extends `ServerTransport` using a filesystem socket path instead of an IP address and port. Creates `UdsSession` instances for each connected client.

```csharp
public class UdsServer : ServerTransport
```

> UDS is only available on Linux and macOS. On Windows, UDS support requires Windows 10 build 17063 or later.

---

## Constructors

```csharp
public UdsServer(string path)                        // e.g. "/tmp/myapp.sock"
public UdsServer(UnixDomainSocketEndPoint endpoint)
```

---

## Socket

```csharp
protected override Socket CreateSocket() 
    => new(Endpoint.AddressFamily, SocketType.Stream, ProtocolType.Unspecified);

protected override void ApplySocketOptions() { } // no-op
```

Uses `SocketType.Stream` with `ProtocolType.Unspecified` (rather than `ProtocolType.Tcp`, since this isn't a TCP socket) over `AddressFamily.Unix`. Standard TCP socket options (keep-alive, no-delay, dual-mode) don't apply to UDS, so `ApplySocketOptions()` is overridden as a no-op.

---

## Session Factory

Override `CreateSession()` in your subclass to use a custom session type:

```csharp
public class MyServer : UdsServer
{
    public MyServer(string path) : base(path) { }

    protected override UdsSession CreateSession() => new MySession(this);
}
```

---

## Inherited API

All server lifecycle, session management, and multicast methods are inherited from `ServerTransport`.

---
