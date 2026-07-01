# PooledObject

**Namespace:** `LuciferCore.Pool`

`PooledObject` is the base for pooled objects in LuciferCore.

It provides:
- object reuse (reduce GC)
- thread-safe reference counting
- safe return-to-pool flow

Common pooled types include:
- `PacketModel`
- `RequestModel`
- `ResponseModel`
- `Buffer`
- `Utf8Builder`

---

## Declarations

```csharp
public abstract class PooledObject
public abstract class PooledObject<T> : PooledObject, IDisposable where T : PooledObject<T>
```

Use:
- `PooledObject<T>` for most cases (has built-in `Dispose()`)
- `PooledObject` only if you need custom disposal behavior

---

## How pool works (simple)

Each type has two levels:

1. thread-local stack (fast path)
2. global concurrent stack (fallback)

Flow:

- `Lucifer.Rent<T>()` -> get instance, set ref count to `1`
- `Lucifer.Return(obj)` / `Dispose()` -> decrement ref count
- when ref count reaches `0` -> call `Reset()` and push back to pool

---

## Main property

| Property | Type | Meaning |
|---|---|---|
| `RefCount` | `int` | Current owner count |

---

## Main methods

| Method | Meaning |
|---|---|
| `SetRefCount(int)` | Set ref count (pool uses this) |
| `IncrementRef()` | Add one owner |
| `DecrementRef()` | Remove one owner |
| `CheckSafety()` | Throw if object already returned/disposed |
| `Reset()` | Clear object for reuse (abstract) |
| `Dispose()` | Return object to pool (`PooledObject<T>` only) |

---

## Share object safely

If two owners use same pooled object, increment ref first:

```csharp
data.IncrementRef(); // owner count +1

Task.Run(() =>
{
    // use data
    Lucifer.Return(data); // -1
});

// current owner done
Lucifer.Return(data); // reaches 0 -> Reset + return pool
```

---

## Subclass template

```csharp
public class MyPooledModel : PooledObject<MyPooledModel>
{
    private byte[]? _data;
    private int _length;

    public ReadOnlySpan<byte> Data
    {
        get
        {
            CheckSafety();
            return _data.AsSpan(0, _length);
        }
    }

    protected internal override void Reset()
    {
        _data = null;
        _length = 0;
    }

    public void Initialize(byte[] source, int length)
    {
        CheckSafety();
        _data = source;
        _length = length;
    }
}
```

Use:

```csharp
using var model = Lucifer.Rent<MyPooledModel>();
model.Initialize(bytes, bytes.Length);
var span = model.Data;
```

---

## Important rules

- Prefer `PooledObject<T>` (built-in `Dispose()`).
- Public state access should call `CheckSafety()`.
- In `Reset()`, work directly with fields (`_field`), not properties.
- Release child resources in `Reset()`.
- Do not use object after `Return()` / after `using` ends.

---

## Notes

- Ref counting uses `Interlocked` (thread-safe).
- `RefCount == 0` means object is in pool.
- `CheckSafety()` helps detect use-after-return early.
- Debug builds may include extra checks for double return bugs.