# Scheduling Primitives

**Namespace:** `LuciferCore.Storage`

LuciferCore provides two scheduling primitives:

- `HeapQueue<T>`: min-heap priority queue
- `Event` / `Event<T>`: scheduled work unit (pool-friendly)

---

## `HeapQueue<T>`

Array-based min-heap.

```csharp
public class HeapQueue<T> where T : IComparable<T>
```

Smallest item always comes first.

### Constructor

```csharp
public HeapQueue(int capacity = 1024)
```

| Parameter | Default | Meaning |
|---|---|---|
| `capacity` | `1024` | initial heap size |

### Properties

| Property | Type | Meaning |
|---|---|---|
| `Count` | `int` | item count |
| `IsEmpty` | `bool` | `Count == 0` |

### Methods

```csharp
void Push(T item)
T Peek()
T Pop()
void Clear()
```

Complexity:
- `Push`: `O(log n)`
- `Pop`: `O(log n)`
- `Peek`: `O(1)`

`Peek`/`Pop` throw if heap is empty.

### Example

```csharp
var queue = new HeapQueue<MyEvent>();

queue.Push(new MyEvent { _tick = 1.5f });
queue.Push(new MyEvent { _tick = 0.3f });
queue.Push(new MyEvent { _tick = 2.0f });

while (!queue.IsEmpty)
{
    var next = queue.Pop(); // 0.3 -> 1.5 -> 2.0
    next.Execute();
}
```

> Not thread-safe. Add your own lock/sync if multi-threaded access.

---

## `Event` (base)

```csharp
public abstract class Event : IComparable<Event>
```

Base scheduled unit.

### Field

| Field | Type | Meaning |
|---|---|---|
| `_tick` | `float` | execution time key |

### Abstract methods

```csharp
public abstract void Execute()
public abstract void Recycle()
```

### Virtual methods

```csharp
public virtual bool Precondition()
protected internal virtual void ExecuteEvent()
protected internal virtual void Cleanup()
```

`CompareTo` uses `_tick` (smaller tick = higher priority).

---

## `Event<T>`

```csharp
public abstract class Event<T> : Event where T : Event<T>
```

Adds type-specific callback/pooling support.

### Extra member

```csharp
public Action<T>? OnExecute;
```

Runs after `Execute()` when precondition passes.

### Pool usage pattern

```csharp
if (!EventPool<MyEvent>.Stack.TryPop(out var ev))
    ev = new MyEvent();

ev._tick = currentTick + delay;
ev.OnExecute = e => HandleResult(e);

queue.Push(ev);
```

After run, return to pool:

```csharp
next.ExecuteEvent();
next.Cleanup();
next.Recycle();
```

---

## Custom event example

```csharp
public class DamageEvent : Event<DamageEvent>
{
    public int Amount;
    public string Target = "";

    public override void Execute()
    {
        // apply damage
    }

    protected internal override void Cleanup()
    {
        Amount = 0;
        Target = "";
    }
}
```

---

## Typical processing loop

```csharp
while (!queue.IsEmpty && queue.Peek()._tick <= currentTick)
{
    var next = queue.Pop();
    next.ExecuteEvent();
    next.Cleanup();
    next.Recycle();
}
```

---

## Notes

- These are generic scheduling primitives, not tied to a specific server type.
- Good fit for manager update loops and background workers.
- `HeapQueue<T>` gives ordered execution with low overhead.
- `Event<T>` helps reduce allocations by pooling event instances.