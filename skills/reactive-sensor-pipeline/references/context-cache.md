# Context Cache

One per pipeline instance. In-memory. Dies with the pipeline.

## Interface

```java
public final class ContextCache {

    private final String sensorId;
    private final Cache<String, Object> cache;

    public ContextCache(String sensorId) {
        this.sensorId = sensorId;
        this.cache = Caffeine.newBuilder()
            .maximumSize(256)
            .build();
    }

    public <T> T get(String key, Class<T> type) {
        Object v = cache.getIfPresent(key);
        if (v == null) return null;
        if (!type.isInstance(v)) throw new IllegalStateException(
            "key=" + key + " expected=" + type.getSimpleName() +
            " actual=" + v.getClass().getSimpleName());
        return type.cast(v);
    }

    public void put(String key, Object value) {
        cache.put(key, value);
    }

    public <T> T computeIfAbsent(String key, Class<T> type, Supplier<T> loader) {
        T existing = get(key, type);
        if (existing != null) return existing;
        T fresh = loader.get();
        cache.put(key, fresh);
        return fresh;
    }

    public String sensorId() { return sensorId; }
}
```

Small on purpose. If you want typed accessors, add them as thin wrappers — don't bloat the base class.

## Keying convention

`<category>.<name>` — `calibration.offset`, `alarm.high_temp_state`, `stream.last_seq`. Enforced by code review, not by the cache.

## What lives here

| Category | Example | Written by | Read by |
|---|---|---|---|
| `calibration.*` | offset, scale, zero-point | pipeline startup, hot reload | stage 3 |
| `alarm.*` | rolling window, last fire time | stage 4 | stage 4 |
| `stream.*` | last sequence, last timestamp | stages 2-5 | stages 2-5 |
| `transport.*` | — **don't** — | — | — |

Transport state stays in stage 1's own fields. The cache is for cross-stage state.

## What does not live here

- Anything that must survive restart → SQLite.
- Anything shared across pipelines → doesn't exist in this architecture. If two sensors truly need shared state, raise it to the Orchestration Manager as an explicit coordination point.
- Configuration → loaded *into* the cache at start, but the source of truth is SQLite.

## Concurrency

All stages for one pipeline run on the same Reactor chain, so most access is effectively single-threaded. The exception is **hot reload** — config updates come from a different scheduler and write to the cache while stages read from it. Caffeine is thread-safe; rely on that.

**Don't read-modify-write without atomics.** For counters and state machines in stage 4, use `cache.get(...)` → compute new value → `cache.put(...)` only if the computation is pure. If another stage could race, use `AtomicReference` wrapped values:

```java
AtomicLong counter = cache.computeIfAbsent("alarm.high_count", AtomicLong.class, AtomicLong::new);
counter.incrementAndGet();
```

## Lifecycle

- **Created:** by `PipelineFactory`, passed to stages 3 and 4 at construction.
- **Populated:** pipeline start loads calibration + rules from SQLite.
- **Refreshed:** admin hot-reload endpoint writes new values.
- **Destroyed:** pipeline stop clears the cache; GC reclaims.

No persistence. No warmup from disk. The cost of rebuilding on restart is acceptable — it's milliseconds of SQLite reads.
