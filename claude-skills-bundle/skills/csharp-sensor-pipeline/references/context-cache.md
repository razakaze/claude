# Context Cache

One per pipeline instance. In-memory. Lives and dies with the pipeline.

## Interface

Hand-rolled `ConcurrentDictionary` wrapper beats `IMemoryCache` for this use case. The cache holds a small, fixed set of keys; eviction is never wanted (eviction would silently break stage 3/4). `IMemoryCache`'s expiration and size tracking are overhead without benefit.

```csharp
public sealed class ContextCache
{
    private readonly ConcurrentDictionary<string, object> _store = new();

    public ContextCache(string sensorId) => SensorId = sensorId;
    public string SensorId { get; }

    public T? Get<T>(string key) where T : class
    {
        if (!_store.TryGetValue(key, out var value)) return null;
        if (value is not T typed)
        {
            throw new InvalidOperationException(
                $"Cache key={key} expected={typeof(T).Name} actual={value.GetType().Name}");
        }
        return typed;
    }

    public T GetRequired<T>(string key) where T : class
        => Get<T>(key) ?? throw new KeyNotFoundException($"Cache key missing: {key}");

    public void Put(string key, object value) => _store[key] = value;

    public T GetOrAdd<T>(string key, Func<T> factory) where T : class
        => (T)_store.GetOrAdd(key, _ => factory());

    public bool Remove(string key) => _store.TryRemove(key, out _);
}
```

For value types on the hot path, wrap in `StrongBox<T>` to avoid boxing on every get/put:

```csharp
public sealed class TypedContextCache(string sensorId)
{
    private readonly ConcurrentDictionary<string, object> _store = new();

    public string SensorId { get; } = sensorId;

    public ref long RefCounter(string key)
    {
        var box = (StrongBox<long>)_store.GetOrAdd(key, _ => new StrongBox<long>(0));
        return ref box.Value;
    }
}
```

Then from stage 4: `Interlocked.Increment(ref cache.RefCounter("alarm.high_count"))`. No boxing, thread-safe.

## Keying convention

`<category>.<n>`:

- `calibration.offset`, `calibration.scale`, `calibration.zero_point`
- `alarm.rules`, `alarm.high_temp_state`, `alarm.last_fire_time`
- `stream.last_seq`, `stream.last_timestamp`

Enforced by code review, not by the cache.

## What lives here

| Category | Example | Written by | Read by |
|---|---|---|---|
| `calibration.*` | offset, scale, zero-point | pipeline startup, hot reload | stage 3 |
| `alarm.*` | rolling window, last-fire time, rules | stage 4 | stage 4 |
| `stream.*` | last seq, last ts | stages 2-5 | stages 2-5 |
| `transport.*` | — **don't** — | — | — |

Transport state stays in stage 1's own fields. The cache is for cross-stage state.

## What does not live here

- Anything that must survive restart → SQLite.
- Anything shared across pipelines → doesn't exist in this architecture. If two sensors genuinely need shared state, elevate it to the Orchestration Manager as explicit coordination.
- Configuration that isn't part of runtime state → stays in SQLite and `SensorConfig`.

## Concurrency

All stages for one pipeline typically run on the same async chain, so access is mostly serialized by the async state machine. Two exceptions:

1. **Hot reload** — admin endpoint writes while stages read. `ConcurrentDictionary` is thread-safe; the read sees either the old or new value, never a half-update.
2. **Channel-separated stages** — producer and consumer run on different threads. Same story; `ConcurrentDictionary` handles it.

For read-modify-write on a cached value (e.g., incrementing a counter), use `Interlocked` on a `StrongBox<T>` as above. Never `var x = cache.Get(...); x.Count++; cache.Put(...)` — races.

## Atomic state machines

Alarm state machines commonly need atomic transitions. Model them as immutable records and use `ConcurrentDictionary.AddOrUpdate` or `Interlocked.CompareExchange` on a `StrongBox<record>`:

```csharp
public readonly record struct HighTempState(int Count, DateTime LastReading);

var box = (StrongBox<HighTempState>)cache.GetOrAdd(
    "alarm.high_temp_state",
    () => new StrongBox<HighTempState>(new HighTempState(0, default)));

HighTempState current, updated;
do
{
    current = box.Value;
    updated = current with { Count = current.Count + 1, LastReading = reading.Timestamp };
} while (Interlocked.CompareExchange(ref box.Value, updated, current) != current);
```

For non-hot-path cases where overhead doesn't matter, a lock is simpler:

```csharp
private readonly object _lock = new();
// ...
lock (_lock)
{
    var state = cache.Get<HighTempState>("alarm.high_temp_state");
    // mutate
    cache.Put("alarm.high_temp_state", newState);
}
```

## Lifecycle

- **Created:** by `PipelineFactory`, passed to stages 3 and 4 at construction.
- **Populated:** pipeline start loads calibration and alarm rules from SQLite.
- **Refreshed:** admin hot-reload endpoint writes new values.
- **Destroyed:** `PipelineInstance` drops its reference on stop; GC reclaims.

No persistence. No warmup from disk. Rebuilding on restart costs milliseconds of SQLite reads — acceptable.

## Observability

Cache access latency is typically sub-microsecond; instrumentation isn't worth it. But if you need to know which keys are being read hot, wrap the cache in a counting decorator during development — don't ship the instrumentation.

## Why not IMemoryCache

- **Eviction policy.** `IMemoryCache` evicts on size or expiration. For a pipeline cache, eviction means stage 3 loses calibration silently — a subtle bug. You can disable eviction, but then you're paying for features you don't use.
- **Allocation.** `IMemoryCache` allocates `MemoryCacheEntry` objects per put. A small `ConcurrentDictionary` doesn't.
- **Complexity.** `IMemoryCache` has options, post-eviction callbacks, priority levels, change tokens. For a fixed small key set, it's overengineering.

Use `IMemoryCache` when eviction is desirable (request caches, HTTP response caches, etc). Don't use it here.
