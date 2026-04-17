# Pipeline Construction

## Stage interfaces

Each stage is a `Function<Flux<In>, Flux<Out>>` with construction-time configuration. Stages are not Spring beans *per sensor* — they're built per pipeline by the factory. The **registries** that resolve stage implementations by sensor type *are* Spring beans.

```java
public interface IntegrationStage {
    boolean supports(TransportConfig transport);
    Flux<byte[]> start(SensorConfig config);
    void stop();
}

public interface ProtocolStage {
    Flux<RawReading> apply(Flux<byte[]> frames);
}

public interface NormalizationStage {
    Flux<CanonicalReading> apply(Flux<RawReading> raw);
}

public interface AlarmStage {
    Flux<SensorEvent> apply(Flux<CanonicalReading> readings);
}

public interface FinalStage {
    Flux<SensorEvent> apply(Flux<SensorEvent> events);  // passthrough + emit to sink
}
```

Domain types (`RawReading`, `CanonicalReading`, `SensorEvent`) are **sensor-family specific** — define them per project. The interfaces stay.

## Registries

A registry resolves a stage implementation for a sensor type. One registry per stage kind.

```java
@Component
public final class ProtocolStageRegistry {

    private final Map<String, ProtocolStageFactory> factories;

    public ProtocolStageRegistry(List<ProtocolStageFactory> beans) {
        this.factories = beans.stream().collect(
            Collectors.toMap(ProtocolStageFactory::protocolName, f -> f));
    }

    public ProtocolStage resolve(String protocol) {
        ProtocolStageFactory f = factories.get(protocol);
        if (f == null) throw new IllegalArgumentException("unknown protocol: " + protocol);
        return f.create();
    }
}

public interface ProtocolStageFactory {
    String protocolName();
    ProtocolStage create();
}
```

Adding a new protocol: implement `ProtocolStageFactory`, mark it `@Component`. Done. No edits to `PipelineFactory`, no edits to the registry.

## Factory

```java
@Component
public final class PipelineFactory {

    private final List<IntegrationStage> integrationStages;
    private final ProtocolStageRegistry protocols;
    private final NormalizationStageRegistry normalizers;
    private final AlarmStageRegistry alarms;
    private final Sinks.Many<SensorEvent> outboundSink;
    private final SensorConfigRepository repo;

    public PipelineInstance build(String sensorId) {
        SensorConfig config = repo.loadEnabled(sensorId)
            .orElseThrow(() -> new IllegalStateException("no config: " + sensorId));

        ContextCache cache = new ContextCache(sensorId);
        loadCalibrationInto(cache, config);
        loadAlarmRulesInto(cache, config);

        IntegrationStage s1 = integrationStages.stream()
            .filter(s -> s.supports(config.transport()))
            .findFirst()
            .orElseThrow(() -> new IllegalStateException(
                "no integration for transport: " + config.transport().type()));

        ProtocolStage      s2 = protocols.resolve(config.protocol());
        NormalizationStage s3 = normalizers.resolve(config.sensorType(), cache);
        AlarmStage         s4 = alarms.resolve(config.sensorType(), cache);
        FinalStage         s5 = new DefaultFinalStage(outboundSink, config);

        Flux<SensorEvent> pipeline = s1.start(config)
            .transform(s2::apply)
            .transform(s3::apply)
            .transform(s4::apply)
            .transform(s5::apply)
            .doOnError(e -> /* metric, log, optional restart */)
            .onErrorResume(e -> Flux.empty());

        return new PipelineInstance(sensorId, pipeline, s1, cache);
    }
}
```

## PipelineInstance

```java
public final class PipelineInstance {

    private final String sensorId;
    private final Flux<SensorEvent> pipeline;
    private final IntegrationStage integration;
    private final ContextCache cache;
    private volatile Disposable subscription;

    public void start() {
        if (subscription != null) return;
        subscription = pipeline.subscribe();
    }

    public void stop() {
        Disposable d = subscription;
        if (d != null) d.dispose();
        integration.stop();
        subscription = null;
    }

    public ContextCache cache() { return cache; }
    public String sensorId() { return sensorId; }
}
```

The pipeline's subscription lives as long as the instance. `stop()` disposes the subscription and tells stage 1 to release the transport.

## PipelineManager

The bean that owns all pipeline instances and reacts to config changes.

```java
@Component
public final class PipelineManager {

    private final PipelineFactory factory;
    private final Map<String, PipelineInstance> running = new ConcurrentHashMap<>();

    @PostConstruct
    public void startAll() {
        for (String id : factory.repo().allEnabledSensorIds()) startOne(id);
    }

    public void startOne(String sensorId) {
        running.computeIfAbsent(sensorId, id -> {
            PipelineInstance p = factory.build(id);
            p.start();
            return p;
        });
    }

    public void stopOne(String sensorId) {
        PipelineInstance p = running.remove(sensorId);
        if (p != null) p.stop();
    }

    public void restartOne(String sensorId) {
        stopOne(sensorId);
        startOne(sensorId);
    }

    public void hotReloadCalibration(String sensorId) {
        PipelineInstance p = running.get(sensorId);
        if (p == null) return;
        factory.loadCalibrationInto(p.cache(), factory.repo().loadEnabled(sensorId).orElseThrow());
    }

    @PreDestroy
    public void stopAll() {
        running.values().forEach(PipelineInstance::stop);
        running.clear();
    }
}
```

Transport/protocol changes go through `restartOne`. Calibration/rule changes go through `hotReloadCalibration` — no restart.

## Outbound sink

One `Sinks.Many<SensorEvent>` for the whole process. Multi-producer, multi-subscriber.

```java
@Configuration
public class SinkConfig {
    @Bean
    public Sinks.Many<SensorEvent> outboundSink() {
        return Sinks.many().multicast().onBackpressureBuffer(1024, false);
    }
}
```

Buffer size 1024 is a starting point — tune to your event rate. The Orchestration Manager subscribes to `sink.asFlux()` and publishes to REST/WebSocket consumers from there.

**`tryEmitNext`, not `emitNext`.** `emitNext` with `Sinks.EmitFailureHandler.FAIL_FAST` throws on backpressure; `tryEmitNext` returns a result enum you can check and drop-with-metric. Drop-with-metric is the right policy for the sensor pipeline — if the Orchestration Manager can't keep up, losing events beats blocking the ingress.
