# Custom HTTP Models

This guide walks through building a custom data model on top of `HttpModel`. You get the full buffer infrastructure — pooled allocation, zero-copy span access, fluent building, and incremental receive parsing — for free. The only thing you define is what your three start-line tokens mean and any domain-specific builder or accessor methods you need.

---

## When to extend `HttpModel`

Extend `HttpModel` when your protocol shares the HTTP wire format — a start-line with exactly three space-separated tokens, followed by `key: value` headers, a blank line, and a body — but the semantics of those tokens differ from a standard request or response.

Practical examples:

- A push notification message: `PUSH /v1/notification/alert HTTP/1.1`
- A server-sent event envelope: `EVENT stream/metrics v2`
- A structured audit log entry: `WRITE /audit/2026 batch/3`
- A proxy command: `CONNECT upstream:8443 HTTP/1.1`

If your protocol does not follow the HTTP start-line + headers + body layout, `HttpModel` is not the right base. Use `PacketModel` or a custom `IRoutable` implementation instead.

---

## Step 1 — Define your token semantics

Decide what each of the three start-line positions will contain in your model. Write them down before writing any code.

```
Example: Push notification message
  FirstSpan  → verb    (e.g. PUSH, PULL, ACK)
  SecondSpan → topic   (e.g. /v1/notification/alert)
  ThirdSpan  → version (e.g. v1)
```

---

## Step 2 — Create the class

Extend `HttpModel` and expose named properties that project onto `FirstSpan`, `SecondSpan`, and `ThirdSpan`:

```csharp
namespace MyApp.Model;

public class PushMessageModel : HttpModel, IRoutable
{
    public PushMessageModel() => Clear();

    // Named projections over the three start-line slots
    public ReadOnlySpan<byte> VerbSpan    => FirstSpan;
    public ReadOnlySpan<byte> TopicSpan   => SecondSpan;
    public ReadOnlySpan<byte> VersionSpan => ThirdSpan;

    // IRoutable — exposes this model to the dispatch pipeline
    public ReadOnlySpan<byte> MethodRoute => FirstSpan;
    public ReadOnlySpan<byte> UrlRoute    => SecondSpan;
}
```

That is the minimum viable custom model. It already supports `SetBegin`, `SetHeader`, `SetBody`, and all inherited read methods.

---

## Step 3 — Add a builder

Override `SetBegin` to provide a domain-typed entry point that calls the base generic implementation:

```csharp
// Default version token
private static ReadOnlySpan<byte> DefaultVersion => "v1"u8;

public PushMessageModel SetBegin(ReadOnlySpan<byte> verb, ReadOnlySpan<byte> topic)
    => SetBegin(verb, topic, DefaultVersion);

public PushMessageModel SetBegin(
    ReadOnlySpan<byte> verb,
    ReadOnlySpan<byte> topic,
    ReadOnlySpan<byte> version)
{
    base.SetBegin(verb, topic, version);
    return this;
}

// Override Clear to reset any extra fields your model adds
public override PushMessageModel Clear()
{
    base.Clear();
    Priority = 0;
    return this;
}
```

Add fluent wrappers over the inherited methods so callers never need to cast:

```csharp
public new PushMessageModel SetHeader<TKey, TValue>(
    ReadOnlySpan<TKey> key,
    ReadOnlySpan<TValue> value)
    where TKey : unmanaged where TValue : unmanaged
{
    base.SetHeader(key, value);
    return this;
}

public new PushMessageModel SetBody<T>(ReadOnlySpan<T> body) where T : unmanaged
{
    base.SetBody(body);
    return this;
}
```

---

## Step 4 — Add quick builders

Compose the chain into named factory methods for the common cases your callers will hit most often:

```csharp
public PushMessageModel MakePushMessage(ReadOnlySpan<byte> topic, ReadOnlySpan<byte> payload)
{
    Clear();
    SetBegin("PUSH"u8, topic);
    SetHeader("Content-Type"u8, "application/octet-stream"u8);
    SetBody(payload);
    return this;
}

public PushMessageModel MakeAckMessage(ReadOnlySpan<byte> topic)
{
    Clear();
    SetBegin("ACK"u8, topic);
    SetBody();
    return this;
}
```

---

## Step 5 — Override cookie and post-parse hooks (optional)

If your protocol uses cookie-like header pairs, override `ParseCookieHeader`:

```csharp
protected override void ParseCookieHeader(ReadOnlySpan<byte> value, ReadOnlySpan<byte> full)
{
    // Parse semicolon-separated key=value pairs from 'value'
    // Record positions using AbsoluteOffset(full, slice) and _cookies.Add(...)
}
```

If you need to extract a derived field after the start-line and headers are fully parsed (e.g. parse `SecondSpan` as an integer), override `OnHeaderParsed`:

```csharp
public int Priority { get; private set; }

protected override void OnHeaderParsed()
{
    // Read a custom "X-Priority" header value and store it
    for (var i = 0; i < Headers; i++)
    {
        if (!TryGetHeader(i, out var key, out var value)) continue;
        if (Lucifer.Equals(key, "X-Priority"u8, true) &&
            System.Buffers.Text.Utf8Parser.TryParse(value, out int p, out _))
        {
            Priority = p;
        }
    }
}
```

---

## Step 6 — Register with the pool and session

Rent instances via `Lucifer.Rent<PushMessageModel>()` and return them with `Lucifer.Return(instance)` or via `using`:

```csharp
using var msg = Lucifer.Rent<PushMessageModel>();
msg.MakePushMessage("/v1/alerts/critical"u8, payloadBytes);
session.Send(msg); // implicit ReadOnlySpan<byte> conversion
```

For receiving, pass the model to `ReceiveHeader` and `ReceiveBody` in your session's receive callbacks — the base class handles all parsing.

---

## Complete example

```csharp
namespace MyApp.Model;

public class PushMessageModel : HttpModel, IRoutable
{
    private static ReadOnlySpan<byte> DefaultVersion => "v1"u8;

    public PushMessageModel() => Clear();

    // Start-line projections
    public ReadOnlySpan<byte> VerbSpan    => FirstSpan;
    public ReadOnlySpan<byte> TopicSpan   => SecondSpan;
    public ReadOnlySpan<byte> VersionSpan => ThirdSpan;

    // IRoutable for dispatch
    public ReadOnlySpan<byte> MethodRoute => FirstSpan;
    public ReadOnlySpan<byte> UrlRoute    => SecondSpan;

    // Extra parsed field
    public int Priority { get; private set; }

    // Clear
    public override PushMessageModel Clear()
    {
        base.Clear();
        Priority = 0;
        return this;
    }

    // SetBegin overloads
    public PushMessageModel SetBegin(ReadOnlySpan<byte> verb, ReadOnlySpan<byte> topic)
        => SetBegin(verb, topic, DefaultVersion);

    public PushMessageModel SetBegin(
        ReadOnlySpan<byte> verb,
        ReadOnlySpan<byte> topic,
        ReadOnlySpan<byte> version)
    {
        base.SetBegin(verb, topic, version);
        return this;
    }

    // Fluent wrappers
    public new PushMessageModel SetHeader<TKey, TValue>(ReadOnlySpan<TKey> key, ReadOnlySpan<TValue> value)
        where TKey : unmanaged where TValue : unmanaged
    { base.SetHeader(key, value); return this; }

    public new PushMessageModel SetBody<T>(ReadOnlySpan<T> body) where T : unmanaged
    { base.SetBody(body); return this; }

    public new PushMessageModel SetBody()
    { base.SetBody(); return this; }

    // Quick builders
    public PushMessageModel MakePushMessage(ReadOnlySpan<byte> topic, ReadOnlySpan<byte> payload)
    {
        Clear();
        SetBegin("PUSH"u8, topic);
        SetHeader("Content-Type"u8, "application/octet-stream"u8);
        SetBody(payload);
        return this;
    }

    public PushMessageModel MakeAckMessage(ReadOnlySpan<byte> topic)
    {
        Clear();
        SetBegin("ACK"u8, topic);
        SetBody();
        return this;
    }

    // Post-parse hook
    protected override void OnHeaderParsed()
    {
        for (var i = 0; i < Headers; i++)
        {
            if (!TryGetHeader(i, out var key, out var value)) continue;
            if (Lucifer.Equals(key, "X-Priority"u8, true) &&
                System.Buffers.Text.Utf8Parser.TryParse(value, out int p, out _))
            {
                Priority = p;
                return;
            }
        }
    }
}
```

Usage:

```csharp
// Building
using var msg = Lucifer.Rent<PushMessageModel>();
msg.MakePushMessage("/v1/alerts/critical"u8, payloadBytes);
session.Send(msg);

// Reading in a handler
[Handler("v1", "push")]
internal class PushHandler : RouteHandler
{
    [WsMessage("alerts")]
    [Safe("")]
    [Authorize(UserRole.Guest)]
    public void OnAlert([Session] MySession session, [Data] PushMessageModel msg)
    {
        var topic    = msg.TopicSpan;
        var payload  = msg.BodySpan;
        var priority = msg.Priority;
    }
}
```

---

## Checklist

Before shipping a custom `HttpModel` subclass:

- Named properties expose `FirstSpan`, `SecondSpan`, `ThirdSpan` with meaningful names.
- `IRoutable` is implemented if the model participates in dispatch.
- `Clear()` is overridden to reset any extra fields added by the subclass.
- Fluent wrappers return the concrete subtype (not `HttpModel`) so callers get a clean API without casts.
- Quick builders call `Clear()` first — consistent with the `RequestModel` / `ResponseModel` convention.
- Any extra parsed fields are populated in `OnHeaderParsed()`, not in property getters.
- Instances are obtained via `Lucifer.Rent<T>()` and released with `using` or `Lucifer.Return`.
