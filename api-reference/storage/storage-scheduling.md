# Scheduling Primitives

**Namespace:** `LuciferCore.Storage`

LuciferCore provides two scheduling primitives for time-ordered event execution: `HeapQueue<T>` — a cache-optimized min-heap, and `Event` / `Event<T>` — a poolable, time-ordered unit of work.

---

## HeapQueue\<T\>

A cache-friendly, array-based **min-heap priority queue**. Items are ordered by their natural `IComparable<T>` ordering — the smallest item is always at the top.

Designed for scheduling scenarios where items must be processed in time order with minimal allocation. The internal array grows by doubling when capacity is exceeded.

```csharp
public class HeapQueue<T> where T : IComparable<T>
```

### Constructor

```csharp
public HeapQueue(int capacity = 1024)
```

| Parameter | Default | Description |
|---|---|---|
| `capacity` | `1024` | Initial array capacity. The array doubles on overflow |

### Properties

| Property | Type | Description |
|---|---|---|
| `Count` | `int` | Number of items currently in the heap |
| `IsEmpty` | `bool` | `true` if `Count == 0` |

### Methods

```csharp
void Push(T item)   // insert item and restore heap order — O(log n)
T    Peek()         // return top item without removing — O(1)
T    Pop()          // remove and return top item — O(log n)
void Clear()        // remove all items
```

`Peek()` and `Pop()` throw `InvalidOperationException` if the heap is empty.

### Usage

```csharp
var queue = new HeapQueue<MyEvent>();

queue.Push(new MyEvent { _tick = 1.5f });
queue.Push(new MyEvent { _tick = 0.3f });
queue.Push(new MyEvent { _tick = 2.0f });

while (!queue.IsEmpty)
{
    var next = queue.Pop(); // returns tick 0.3, then 1.5, then 2.0
    next.Execute();
}
```

### Remarks

- Heap operations use integer bit-shifting for parent/child index calculation — no division overhead.
- Popped items have their array slot set to `default` to release object references and avoid memory leaks.
- Not thread-safe. Use external synchronization if accessed from multiple threads.

---

## Event (base class)

`Event` is the abstract base for all scheduled units of work. It implements `IComparable<Event>` by comparing `_tick` values, making it directly usable with `HeapQueue<Event>`.

```csharp
public abstract class Event : IComparable<Event>
```

### Field

| Field | Type | Description |
|---|---|---|
| `_tick` | `float` (internal) | Scheduled simulation time for this event. Set before pushing to `HeapQueue` |

### Abstract Members

```csharp
public abstract void Execute()   // event logic — implement in your subclass
public abstract void Recycle()   // return this instance to the pool
```

### Virtual Members

```csharp
public virtual bool Precondition()               // return false to skip execution. Default: true
protected internal virtual void ExecuteEvent()   // checks Precondition(), then calls Execute()
protected internal virtual void Cleanup()        // called after execution for cleanup. Default: no-op
```

### Ordering

`CompareTo` compares `_tick` values — lower tick = higher priority = processed first by `HeapQueue`.

---

## Event\<T\> (generic base)

```csharp
public abstract class Event<T> : Event where T : Event<T>
```

Extends `Event` with a static callback and type-specific pooling via `EventPool<T>`.

### Additional Member

```csharp
public Action<T>? OnExecute;
```

Invoked after `Execute()` if `Precondition()` passes. Useful for chaining or result handling without subclassing.

### Pooling

`Recycle()` pushes `this` onto `EventPool<T>.Stack` (a `ConcurrentStack<T>`). Rent a pooled instance by popping from `EventPool<T>.Stack`:

```csharp
if (!EventPool<MyEvent>.Stack.TryPop(out var ev))
    ev = new MyEvent();

ev._tick = currentTick + delay;
ev.OnExecute = e => HandleResult(e);

queue.Push(ev);
```

After `Pop()` from the heap and `ExecuteEvent()`, call `ev.Recycle()` to return it to the pool.

### Implementing a Custom Event

```csharp
public class DamageEvent : Event<DamageEvent>
{
    public int Amount;
    public string Target = "";

    public override void Execute()
    {
        // apply damage to Target
    }

    protected internal override void Cleanup()
    {
        Amount = 0;
        Target = "";
    }
}
```

```csharp
// Scheduling
var ev = new DamageEvent { _tick = 5.0f, Amount = 100, Target = "boss" };
queue.Push(ev);

// Processing loop
while (!queue.IsEmpty && queue.Peek()._tick <= currentTick)
{
    var next = queue.Pop();
    next.ExecuteEvent();
    next.Cleanup();
    next.Recycle();
}
```

---

## Remarks

- `Event` and `HeapQueue<T>` are **general-purpose scheduling primitives** — they are not tied to any specific server or session lifecycle. Use them wherever tick-based or time-ordered execution is needed within a Manager's `Update()` loop or a background service.
- `EventPool<T>` is `internal` and not directly accessible from application code. Use `Recycle()` / `EventPool<T>.Stack.TryPop()` through the `Event<T>` API as shown above.
