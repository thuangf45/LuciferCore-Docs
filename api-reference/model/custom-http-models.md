# Custom HTTP Models

This guide shows how to build a custom model from `HttpModel<T>`.

You keep built-in features:
- pooled memory
- zero-copy spans
- fluent builder API
- HTTP-style parsing

You only define:
- what 3 start-line tokens mean
- any extra helper fields/methods

---

## When to use `HttpModel<T>`

Use it when your protocol still follows HTTP shape:

1. start-line with 3 tokens
2. headers (`key: value`)
3. blank line
4. body

If your protocol is not this shape, use another model (like `PacketModel` or custom `IRoutable`).

> Important: inherit from `HttpModel<T>`, not plain `HttpModel`, if you want fluent methods like `SetBegin`, `SetHeader`, `SetBody`.

---

## Step 1: define token meaning

Example:

```text
FirstSpan  -> Verb     (PUSH, ACK, ...)
SecondSpan -> Topic    (/v1/notification/alert)
ThirdSpan  -> Version  (v1)
```

---

## Step 2: create class

```csharp
public class PushMessageModel : HttpModel<PushMessageModel>, IRoutable
{
    public PushMessageModel() => Clear();

    public ReadOnlySpan<byte> VerbSpan    => FirstSpan;
    public ReadOnlySpan<byte> TopicSpan   => SecondSpan;
    public ReadOnlySpan<byte> VersionSpan => ThirdSpan;

    public ReadOnlySpan<byte> MethodRoute => FirstSpan;
    public ReadOnlySpan<byte> UrlRoute    => SecondSpan;
}
```

Now you already get fluent builder methods from base class.

---

## Step 3: add helper overloads

You can add convenience overloads, for example default version:

```csharp
private static ReadOnlySpan<byte> DefaultVersion => "v1"u8;

public PushMessageModel SetBegin(ReadOnlySpan<byte> verb, ReadOnlySpan<byte> topic)
    => SetBegin(verb, topic, DefaultVersion);
```

If you add custom fields, reset them in `Clear()` (hide with `new`):

```csharp
public int Priority { get; private set; }

public new PushMessageModel Clear()
{
    base.Clear();
    Priority = 0;
    return this;
}
```

---

## Step 4: add quick builders

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

## Step 5: optional parse hooks

Override virtual hooks when needed:

- `ParseCookieHeader(...)`
- `OnHeaderParsed()`

Example parse extra header field:

```csharp
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
```

---

## Step 6: use with pool

```csharp
using var msg = Lucifer.Rent<PushMessageModel>();
msg.MakePushMessage("/v1/alerts/critical"u8, payloadBytes);
session.Send(msg);
```

---

## Full example

```csharp
public class PushMessageModel : HttpModel<PushMessageModel>, IRoutable
{
    private static ReadOnlySpan<byte> DefaultVersion => "v1"u8;

    public PushMessageModel() => Clear();

    public ReadOnlySpan<byte> VerbSpan    => FirstSpan;
    public ReadOnlySpan<byte> TopicSpan   => SecondSpan;
    public ReadOnlySpan<byte> VersionSpan => ThirdSpan;

    public ReadOnlySpan<byte> MethodRoute => FirstSpan;
    public ReadOnlySpan<byte> UrlRoute    => SecondSpan;

    public int Priority { get; private set; }

    public new PushMessageModel Clear()
    {
        base.Clear();
        Priority = 0;
        return this;
    }

    public PushMessageModel SetBegin(ReadOnlySpan<byte> verb, ReadOnlySpan<byte> topic)
        => SetBegin(verb, topic, DefaultVersion);

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

---

## Quick checklist

- Inherit from `HttpModel<T>` (CRTP), not plain `HttpModel`
- Map `FirstSpan/SecondSpan/ThirdSpan` to clear names
- Implement `IRoutable` if dispatch needs this model
- Use `new Clear()` to reset custom fields
- Override only real hooks (`ParseCookieHeader`, `OnHeaderParsed`) when needed
- Rent/return instances via pool (`Lucifer.Rent<T>()`, `using`)