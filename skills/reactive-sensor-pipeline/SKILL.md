---
name: java-reactive-sensor-pipeline
description: Use when writing or modifying Java 17 Spring WebFlux code that ingests data from physical sensors (UDP, TCP, serial/COM, Bluetooth Classic, BLE) through a Reactor pipeline and publishes to an in-process Orchestration Manager. Triggers on mentions of Reactor Netty, Flux/Mono sensor streams, jSerialComm, BlueZ, hid4java, GATT, sensor calibration, alarm thresholds, YAML-configured pipelines, SQLite sensor configuration, or the five-stage pipeline (integration → protocol → normalization → alarm → final). Use whenever the user is building a pipeline-runner service in this codebase, adding a new sensor type, wiring a new transport, changing stage contracts, or optimizing sensor-to-response p99 latency. Consult even when the user doesn't name these terms — any code that reads a sensor and produces an event in this project is in scope.
---

# Reactive Sensor Pipeline

Build sensor ingestion as a **five-stage Reactor pipeline**, one instance per sensor, configured from SQLite (seeded from YAML), with an in-memory context cache per pipeline, publishing POJOs to an in-process Orchestration Manager via a `Sinks.Many`.

The five stages are fixed. Everything else varies per sensor and lives in config, not in code.

---

## Architecture at a glance

```
YAML (seed) ──► SQLite (runtime truth) ──► PipelineFactory
                                                  │
                                                  ▼
                               ┌─ PipelineInstance (per sensor) ─┐
                               │  ContextCache (in-memory)        │
                               │  [1] Integration                 │
                               │  [2] Protocol                    │
                               │  [3] Normalization               │
                               │  [4] Alarm                       │
                               │  [5] Final ──┐                   │
                               └──────────────┼───────────────────┘
                                              ▼
                                    Sinks.Many<SensorEvent>
                                              ▼
                               ┌─ Orchestration Manager ────────┐
                               │  REST API (WebFlux)            │
                               │  OAuth2 Resource Server        │
                               │  Subscribes to the Sink        │
                               └────────────────────────────────┘
```

The pipeline runner and Orchestration Manager are **one deployable**, same JVM. No network between them. No auth between them. The boundary is an interface, not a socket.

---

## The five stages — contract

Every pipeline flows through these stages, in this order. Nothing else. If a stage does another stage's job, the abstraction is wrong — fix the stage, don't add a sixth.

### Stage 1 — Integration
Transport-level I/O. Input: the wire. Output: `Flux<byte[]>`, one emission per logical frame.

- **Allowed:** socket/port setup, reconnect logic, framing (length-prefix, delimiter, timeout), backpressure strategy selection.
- **Forbidden:** decoding payload semantics, applying calibration, evaluating rules, touching the context cache for anything other than connection state.
- **Lifecycle:** must expose `start()` and `stop()`. `stop()` must release the transport within 2s.

### Stage 2 — Protocol
Parse bytes into a typed raw reading. Input: `Flux<byte[]>`. Output: `Flux<RawReading>` where `RawReading` is a `record` specific to the sensor protocol (not the physical sensor).

- **Allowed:** decoding, checksum validation, malformed-frame rejection (drop with metric increment, don't throw).
- **Forbidden:** unit conversion, calibration, cross-frame state beyond what framing requires.
- **Error policy:** a malformed frame logs at DEBUG, increments `malformed_frames_total{sensor=...}`, and is dropped. Does not terminate the `Flux`.

### Stage 3 — Normalization
Convert raw readings into a canonical domain reading. Input: `Flux<RawReading>`. Output: `Flux<CanonicalReading>`.

- **Allowed:** unit conversion, timestamp normalization (to UTC `Instant`), coordinate frame transforms, per-sensor calibration pulled from the context cache.
- **Forbidden:** threshold evaluation, alarm state, any decision that depends on the *history* of readings.
- **Calibration lookup:** read from context cache only. Never hit SQLite on the hot path.

### Stage 4 — Alarm
Stateful rule evaluation. Input: `Flux<CanonicalReading>`. Output: `Flux<SensorEvent>` where `SensorEvent` carries the reading plus zero or more `AlarmAnnotation`s.

- **Allowed:** rolling windows, threshold state machines, debouncing, reads *and writes* to the context cache.
- **Forbidden:** delivery concerns, I/O, blocking operations.
- **State scope:** all alarm state lives in the context cache. Do not use static fields, do not use stage-local fields except for configuration set once at construction.

### Stage 5 — Final
Hand the event off. Input: `Flux<SensorEvent>`. Output: `Mono<Void>` per event, completing when the event has been accepted by the Orchestration Manager's sink.

- **Allowed:** emit to `Sinks.Many<SensorEvent>`, metric increments, final-mile filtering (e.g. suppress events with no annotations if the sensor is configured "alarms-only").
- **Forbidden:** transforming the event payload (that was stage 3's job), adding annotations (stage 4's), persisting to storage directly.

---

## Transport recipes (stage 1)

Each transport has a native shape in the Reactor world. Picking the wrong bridge adds 10–50ms of avoidable p99 latency. Match the transport to the recipe.

| Transport | Native shape | Bridging | Scheduler |
|---|---|---|---|
| UDP | Reactor Netty `UdpClient`/`UdpServer` | None — stay in event loop | `Schedulers.immediate()` |
| TCP | Reactor Netty `TcpClient`/`TcpServer` | None — stay in event loop | `Schedulers.immediate()` |
| Serial (jSerialComm) | Blocking `InputStream` | Dedicated single thread per port, `Sinks.Many` | Dedicated `Scheduler` |
| Bluetooth Classic (RFCOMM) | Blocking `InputStream` | Same as serial | Dedicated `Scheduler` |
| BLE (GATT notifications) | Callback-based | `Flux.create` with `FluxSink` | `Schedulers.immediate()` — callback already off the hot thread |

**Never use `Schedulers.boundedElastic()` on the sensor ingress path.** Its scheduling jitter ruins p99 latency for sensors with tight budgets. It's fine in stage 5 if the Orchestration Manager's sink is bounded and you need overflow handling, but default to `immediate` first.

For detailed code per transport, see `references/transports.md`.

---

## Configuration: YAML → SQLite → runtime

**YAML is seed. SQLite is truth.**

On startup, if a sensor configuration exists in YAML but not in SQLite, it's inserted. If both exist, **SQLite wins** — never overwrite runtime config with seed. A `--reseed` flag or equivalent admin endpoint is the only way YAML overwrites SQLite.

**Schema shape (not prescriptive, but follow unless there's a reason not to):**
- `sensor` — one row per sensor instance (id, type, enabled, transport config as JSON).
- `sensor_protocol_config` — protocol-specific parameters, FK to `sensor`.
- `sensor_calibration` — normalization parameters, FK to `sensor`. Hot-reloadable.
- `sensor_alarm_rule` — rules per sensor, FK to `sensor`. Hot-reloadable.

**Hot reload:** calibration and alarm rules must be reloadable without restarting the pipeline. Integration and protocol config changes require a pipeline restart — stop the instance, rebuild, start. Don't try to mutate a running transport.

SQLite access uses R2DBC (`r2dbc-sqlite`) for reactive reads, or JDBC on a bounded scheduler for writes that happen off the hot path (config changes, seeding). Never JDBC from a pipeline stage.

For schema migration and the YAML→SQLite seeder, see `references/config.md`.

---

## Context cache

One `ContextCache` per pipeline instance. In-memory. Lives and dies with the pipeline.

**Use `Caffeine`** unless the cache is so small and static that `ConcurrentHashMap` is obviously enough (fewer than ~20 keys, no expiration needed).

**What goes in the cache:**
- Calibration parameters loaded from SQLite at pipeline start, refreshed on hot reload.
- Alarm state machines (rolling windows, debounce counters, last-alarm timestamps).
- Last-seen timestamp and sequence number for gap detection.

**What does not go in the cache:**
- Transport connection state — that lives in stage 1's own fields.
- Anything that must survive a pipeline restart — that goes in SQLite.
- Cross-pipeline state — caches don't share. If two sensors need shared state, that's an Orchestration Manager concern.

**Access pattern:** stages receive the `ContextCache` via constructor injection at pipeline build time. Stages never look up the cache by sensor id — the cache they hold *is* the one for their sensor.

For `ContextCache` interface and usage, see `references/context-cache.md`.

---

## Pipeline construction

A `PipelineFactory` reads one row from SQLite, builds the five stages, wires them into a `Flux`, and returns a `PipelineInstance` with `start()` and `stop()`. One factory call per sensor.

Stage construction uses **Spring's `ObjectProvider<T>`** pattern for transport handlers, so adding a new transport is adding a bean — not editing the factory. The factory picks the bean by transport type from the sensor's config.

Rough shape, for orientation:

```java
public final class PipelineFactory {
    private final ObjectProvider<IntegrationStage> integrationStages;
    private final ProtocolStageRegistry protocols;
    private final NormalizationStageRegistry normalizers;
    private final AlarmStageRegistry alarms;
    private final Sinks.Many<SensorEvent> outboundSink;

    public PipelineInstance build(SensorConfig config) {
        ContextCache cache = new ContextCache(config.sensorId());
        IntegrationStage s1 = integrationStages.stream()
            .filter(s -> s.supports(config.transport()))
            .findFirst().orElseThrow();
        ProtocolStage    s2 = protocols.resolve(config.protocol());
        NormalizationStage s3 = normalizers.resolve(config.sensorType(), cache);
        AlarmStage       s4 = alarms.resolve(config.sensorType(), cache);
        FinalStage       s5 = new FinalStage(outboundSink, config);

        Flux<SensorEvent> pipeline = s1.start(config)
            .transform(s2::apply)
            .transform(s3::apply)
            .transform(s4::apply)
            .transform(s5::apply);

        return new PipelineInstance(config.sensorId(), pipeline, s1);
    }
}
```

Each stage's `apply` is `Function<Flux<In>, Flux<Out>>`. That's the stage contract.

For the full factory, stage interfaces, and lifecycle handling, see `references/pipeline-construction.md`.

---

## Latency budgets

Per-sensor, not global. Each sensor's YAML declares `p99_budget_ms`. The pipeline measures end-to-end latency from stage 1 emission to `Sinks.Many.tryEmitNext` return and emits a metric `sensor_latency_ms{sensor=..., quantile=0.99}`.

**Budget violations are not exceptions.** They are metric signals. An alarm rule on the metric is the right place to react, not a `doOnNext` that throws.

**Allocation discipline on the hot path:**
- No `stream()` on collections with fewer than ~16 elements — use a for-loop. JIT inlines the loop; `stream()` allocates.
- Reuse `ByteBuf` slices via `retainedSlice()`, release in stage 2 after parsing.
- `record` types for readings are fine — modern JVMs stack-allocate short-lived records via scalar replacement.
- Avoid `Flux.zip` on the hot path unless you need true pairing — use `Flux.combineLatest` or merge at stage boundaries instead.

---

## Security

Two surfaces, only one matters here.

**Pipeline runner → Orchestration Manager:** no auth. Same JVM, internal interface, `Sinks.Many` hand-off. Don't add auth here "for defense in depth" — the defense is the process boundary, which doesn't exist.

**Client → Orchestration Manager REST:** Spring Security reactive chain with OAuth2 resource server validating JWTs. Configured in the Orchestration Manager module only. The pipeline runner module has zero Spring Security dependencies.

**Input validation inside the pipeline:**
- Stage 2 rejects malformed frames (metric, drop, continue).
- Stage 3 rejects readings outside physical plausibility (e.g. -300°C temperature) — same policy, metric + drop.
- Stage 4 is where business-rule alarms fire, not where validation happens.

For the Orchestration Manager's Spring Security config and input validation patterns, see `references/security.md`.

---

## Testing

- **Stage unit tests:** each stage is `Function<Flux<In>, Flux<Out>>`. Test with `StepVerifier`, feed synthetic input. No Spring context. No transport. No SQLite.
- **Transport tests:** a fake transport (in-memory `Sinks.Many<byte[]>`) drives stage 1 from the test side. Real transports are integration-tested against loopback/emulators, not unit tests.
- **Pipeline tests:** wire five real stages with a fake transport and an in-memory outbound sink. Assert events landing on the sink.
- **Config tests:** YAML → SQLite seeder runs against an in-memory SQLite (`:memory:`), asserts row presence and the SQLite-wins rule.

No test touches real hardware. Hardware is validated in on-device smoke tests, separately.

For `StepVerifier` patterns and fake-transport helpers, see `references/testing.md`.

---

## When you hit something this skill doesn't cover

The five stages, the YAML→SQLite relationship, the per-pipeline cache, the in-process hand-off to the Orchestration Manager, and the no-`boundedElastic`-on-ingress rule are **load-bearing**. Don't relax them to make a new requirement fit — flag the conflict.

Anything else (specific Reactor operators, specific SQLite schema, specific alarm rule DSL) is a local decision. Make it, follow the CLAUDE.md rules, move on.
