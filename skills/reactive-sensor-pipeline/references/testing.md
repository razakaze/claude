# Testing

No test touches real hardware. Real hardware is validated separately in on-device smoke tests. Everything here is JVM-only.

## Stage unit tests

Each stage is `Function<Flux<In>, Flux<Out>>`. Test with `StepVerifier`. No Spring context.

```java
class TemperatureProtocolStageTest {

    @Test
    void parsesValidFrame() {
        ProtocolStage stage = new TemperatureProtocolStage();
        byte[] frame = HexFormat.of().parseHex("aa0123047f00");

        StepVerifier.create(stage.apply(Flux.just(frame)))
            .assertNext(r -> {
                assertThat(r.sensorId()).isEqualTo("0123");
                assertThat(r.rawValue()).isEqualTo(1151);
            })
            .verifyComplete();
    }

    @Test
    void dropsFrameWithBadChecksum() {
        ProtocolStage stage = new TemperatureProtocolStage();
        byte[] bad = HexFormat.of().parseHex("aa0123047f01");

        StepVerifier.create(stage.apply(Flux.just(bad)))
            .verifyComplete();  // dropped, not errored
    }
}
```

Verify **drops** by asserting the output completes without emitting. Never assert on log output.

## Stateful stage tests (stage 4)

Alarm stages read/write the context cache. Test with a real `ContextCache` — it's just Caffeine, no need to mock.

```java
@Test
void firesAfterThreeConsecutiveHighReadings() {
    ContextCache cache = new ContextCache("s1");
    AlarmStage stage = new HighTemperatureAlarmStage(cache, thresholdCelsius(80));

    CanonicalReading r1 = reading(81);
    CanonicalReading r2 = reading(82);
    CanonicalReading r3 = reading(83);

    StepVerifier.create(stage.apply(Flux.just(r1, r2, r3)))
        .assertNext(e -> assertThat(e.annotations()).isEmpty())
        .assertNext(e -> assertThat(e.annotations()).isEmpty())
        .assertNext(e -> assertThat(e.annotations())
            .extracting(AlarmAnnotation::code)
            .containsExactly("HIGH_TEMP"))
        .verifyComplete();
}
```

## Fake transport

A single helper, reused across integration-stage tests.

```java
public final class FakeIntegrationStage implements IntegrationStage {

    private final Sinks.Many<byte[]> sink = Sinks.many().multicast().onBackpressureBuffer();

    public boolean supports(TransportConfig t) { return t.type().equals("fake"); }
    public Flux<byte[]> start(SensorConfig config) { return sink.asFlux(); }
    public void stop() { sink.tryEmitComplete(); }

    public void push(byte[] frame) {
        Sinks.EmitResult r = sink.tryEmitNext(frame);
        if (r.isFailure()) throw new IllegalStateException("emit failed: " + r);
    }
}
```

Lets pipeline tests drive input deterministically.

## End-to-end pipeline test

Five real stages, fake transport, in-memory outbound sink.

```java
@Test
void fullPipelineEmitsAnnotatedEvent() {
    FakeIntegrationStage s1 = new FakeIntegrationStage();
    ProtocolStage        s2 = new TemperatureProtocolStage();
    ContextCache cache = new ContextCache("s1");
    cache.put("calibration.offset", 0.0);
    NormalizationStage   s3 = new TemperatureNormalizationStage(cache);
    AlarmStage           s4 = new HighTemperatureAlarmStage(cache, 80);
    Sinks.Many<SensorEvent> out = Sinks.many().unicast().onBackpressureBuffer();
    FinalStage           s5 = new DefaultFinalStage(out, sensorConfig());

    Flux<SensorEvent> pipeline = s1.start(sensorConfig())
        .transform(s2::apply)
        .transform(s3::apply)
        .transform(s4::apply)
        .transform(s5::apply);

    Disposable sub = pipeline.subscribe();

    StepVerifier.create(out.asFlux().take(1))
        .then(() -> s1.push(frameAt(85)))
        .then(() -> s1.push(frameAt(86)))
        .then(() -> s1.push(frameAt(87)))
        .assertNext(e -> assertThat(e.annotations()).isNotEmpty())
        .verifyComplete();

    sub.dispose();
}
```

## Config tests

In-memory SQLite (`jdbc:sqlite::memory:`). Run Flyway migrations, run the seeder, assert rows.

```java
@Test
void sqliteWinsOnConflict() {
    DataSource ds = inMemorySqlite();
    runMigrations(ds);
    insertSensorDirectly(ds, "s1", /* transport */ "udp", /* port */ 5000);

    new YamlSeeder().seed(yaml("""
        sensors:
          - sensorId: s1
            transport: { type: udp, port: 9999 }
        """), ds, false);

    SensorConfig loaded = loadSensor(ds, "s1");
    assertThat(loaded.transport().asUdp().port()).isEqualTo(5000);  // SQLite won
}
```

## Scheduler caveat

Many stages use `Schedulers.immediate()` and run synchronously in tests. When a stage uses a dedicated scheduler (serial, Bluetooth), `StepVerifier` needs `.thenAwait(Duration.ofMillis(...))` or `VirtualTimeScheduler` for deterministic timing. Prefer virtual time over real sleeps.

```java
StepVerifier.withVirtualTime(() -> stage.apply(source))
    .thenAwait(Duration.ofSeconds(1))
    .expectNextCount(10)
    .verifyComplete();
```

## What not to test

- Reactor's own behavior. `Flux.map` works; don't test it.
- Spring's DI wiring. `@SpringBootTest` for the whole context is slow and brittle; prefer slice tests or plain JUnit with hand-wired beans.
- Transport libraries' internals. If jSerialComm has a bug, that's upstream.

## What to test hard

- Frame parsing edge cases (short reads, split frames, trailing garbage).
- Alarm state machines across restart boundaries (if the pipeline restarts, does the alarm re-fire? Should it?).
- Backpressure: what happens when the outbound sink is full. Assert the drop policy.
- YAML→SQLite seeder, especially conflict handling and `--reseed`.
