# PooledObject

**Namespace:** `LuciferCore.Pool`

`PooledObject` is the abstract base class for all objects managed by LuciferCore's internal object pool. It provides thread-safe reference counting so pooled objects can be safely shared across async contexts before being returned.

All major LuciferCore model and utility types extend `PooledObject`: `PacketModel`, `RequestModel`, `ResponseModel`, `Buffer`, `Utf8Builder`, and more.

---

## How the Pool Works

LuciferCore maintains a **two-tier pool** per type:

1. **Thread-local stack** (up to 32 objects per thread) — checked first. Zero contention, near-zero overhead.
2. **Global concurrent stack** — fallback when the thread-local stack is empty, or overflow target when it is full.

When you call `Lucifer.Rent<T>()`, the pool pops an existing instance, sets its reference count to `1`, and returns it. A freshly constructed `PooledObject` starts with `_refCount = 1`, meaning it is immediately in an active, owned state without requiring an explicit `SetRefCount` call at construction time. When you call `Lucifer.Return(obj)` (or `Dispose()`), the reference count is decremented — if it reaches `0`, `Reset()` is called and the object is pushed back to the pool.

This design means **no GC pressure** for hot-path objects: the same instances are reused indefinitely across requests.

---

## Declaration

```csharp
public abstract class PooledObject
```

---

## Properties

| Property | Type | Description |
|---|---|---|
| `RefCount` | `int` | Current reference count. Managed atomically via `Interlocked` |

---

## Methods

### `SetRefCount(int count)`

```csharp
public void SetRefCount(int count)
```

Sets the reference count to an explicit value using `Interlocked.Exchange`. Called automatically by the pool when renting — sets the count to `1`. You should not need to call this directly.

---

### `IncrementRef()`

```csharp
public int IncrementRef()
```

Atomically increments the reference count and returns the new value. Use this when sharing a pooled object across multiple owners — for example, passing a `PacketModel` to a background task while the handler method also reads it.

```csharp
data.IncrementRef(); // now RefCount == 2
Task.Run(() =>
{
    // process data...
    Lucifer.Return(data); // decrements to 1 — not returned yet
});
// handler finishes:
Lucifer.Return(data); // decrements to 0 — returned to pool
```

---

### `DecrementRef()`

```csharp
public int DecrementRef()
```

Atomically decrements the reference count and returns the new value. Called by `Lucifer.Return()` — if the result is `0`, `Reset()` is invoked and the object is returned to the pool. Negative counts indicate a double-return bug.

---

### `CheckSafety()`

```csharp
protected void CheckSafety()
```

Validates that the object is still in an active, rented state (`RefCount > 0`). The implementation has two layers:

1. **`DEBUG` builds:** `Debug.Assert(RefCount > 0)` — surfaces the violation immediately in the debugger with a descriptive message before the exception is thrown.
2. **All builds:** throws `ObjectDisposedException` via a `[DoesNotReturn]` helper if `_refCount <= 0`, allowing the JIT to better optimize the happy path (the throw path is never inlined).

Called internally by subclasses on every access to protected state. For example, `Utf8Builder.Cache` and `PacketModel.Buffer` call `CheckSafety()` in both getter and setter before touching internal buffers.

```csharp
// Will throw ObjectDisposedException if called after Dispose():
builder.Dispose();
builder.Append("oops"u8); // ← throws: "Utf8Builder is already disposed or not rented from pool"
```

Do not call `CheckSafety()` in your own application code — it is an internal guard for `PooledObject` subclass implementations.

---

### `Reset()` *(abstract, internal)*

```csharp
protected internal abstract void Reset()
```

Called by the pool when the reference count reaches `0`. Clears all internal state so the object is ready for reuse. Each concrete class implements this to return its own sub-resources (e.g. `Buffer`, cached data) back to their respective pools.

Do not call `Reset()` directly.

---

## Subclassing

If you need a custom type that participates in LuciferCore's pool, extend `PooledObject` and follow the four-section pattern below. This is the same pattern used internally by `PacketModel`, `RequestModel`, `ResponseModel`, `Buffer`, and `Utf8Builder`.

---

### Section 1 — Fields (`_` prefix)

Declare all mutable state as **private fields with an underscore prefix**. These are the "back door" — they bypass `CheckSafety()` and are used exclusively inside `Reset()` to clean up state after the object is returned to the pool.

```csharp
private byte[]? _data;
private int _length;
```

> **Rule:** `Reset()` must only access `_fields` directly, never through properties. At the point `Reset()` is called, `RefCount` may already be `0` — calling any property that invokes `CheckSafety()` would throw `ObjectDisposedException` inside the pool's own cleanup path.

---

### Section 2 — Properties (the "front door")

Every **public property** that exposes internal state must call `CheckSafety()` first. This is the safety contract: all external access goes through the front door.

```csharp
public ReadOnlySpan<byte> Data
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    get
    {
        CheckSafety(); // throws ObjectDisposedException if RefCount <= 0
        return _data.AsSpan(0, _length);
    }
}
```

---

### Section 3 — Lifecycle (`Reset` + `Dispose`)

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
protected internal override void Reset()
{
    // 1. Release any sub-resources (child pooled objects, rented arrays, etc.)
    //    If _data was rented from ArrayPool, return it here.

    // 2. Clear all fields back to defaults
    _data = null;
    _length = 0;

    // IMPORTANT: Never call CheckSafety() or any property that calls it here.
    // RefCount is 0 at this point — the pool owns the object.
}

[MethodImpl(MethodImplOptions.AggressiveInlining)]
public void Dispose()
{
    Lucifer.Return(this); // decrements RefCount → 0 → triggers Reset() → returned to pool
    GC.SuppressFinalize(this); // prevent GC from calling finalizer — Zero-GC
}
```

---

### Section 4 — Business Logic

Any method that operates on the object's state must call `CheckSafety()` before touching fields:

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public void Initialize(byte[] source, int length)
{
    CheckSafety(); // guard before any mutation
    _data = source;
    _length = length;
}
```

---

### Complete Template

```csharp
public class MyPooledModel : PooledObject, IDisposable
{
    // ── Section 1: Fields (back door — used by Reset() only) ──────────────
    private byte[]? _data;
    private int _length;

    // ── Section 2: Properties (front door — CheckSafety on every access) ──
    public ReadOnlySpan<byte> Data
    {
        [MethodImpl(MethodImplOptions.AggressiveInlining)]
        get
        {
            CheckSafety();
            return _data.AsSpan(0, _length);
        }
    }

    // ── Section 3: Lifecycle ───────────────────────────────────────────────
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    protected internal override void Reset()
    {
        // Release sub-resources first, then null fields
        _data = null;
        _length = 0;
        // Never call CheckSafety() or any property here
    }

    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Dispose()
    {
        Lucifer.Return(this);
        GC.SuppressFinalize(this);
    }

    // ── Section 4: Business logic ─────────────────────────────────────────
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    public void Initialize(byte[] source, int length)
    {
        CheckSafety();
        _data = source;
        _length = length;
    }
}
```

### Renting and returning

```csharp
// Rent
using var model = Lucifer.Rent<MyPooledModel>();
model.Initialize(bytes, bytes.Length);

// Use
var span = model.Data;

// Return (automatic via using, or explicit)
Lucifer.Return(model);
```

---

### Summary of Rules

| Rule | Reason |
|---|---|
| Public properties must call `CheckSafety()` | Catch use-after-dispose immediately |
| `Reset()` must only access `_fields`, never properties | `RefCount` is `0` when `Reset()` runs — properties would throw |
| Release sub-resources inside `Reset()`, not `Dispose()` | `Dispose()` just calls `Lucifer.Return(this)`, which triggers `Reset()` |
| Always call `GC.SuppressFinalize(this)` in `Dispose()` | Prevents the GC finalizer from running — keeps the pipeline Zero-GC |

---

## Remarks

- Reference counting is implemented with `Interlocked` operations — all methods are thread-safe and lock-free.
- A `RefCount` of `0` means the object is currently in the pool and not owned by anyone. A value of `1` means it has exactly one owner. Values above `1` mean it is shared.
- `CheckSafety()` throws `ObjectDisposedException` when `RefCount <= 0`. Subclasses call it on every property access to detect use-after-dispose immediately rather than causing silent data corruption.
- Never access a pooled object after calling `Lucifer.Return()` or after the `using` block ends — the instance may have been reset and re-rented by another caller.
- In `DEBUG` builds, the pool layer includes additional assertions to catch double-returns and use-after-return bugs.
