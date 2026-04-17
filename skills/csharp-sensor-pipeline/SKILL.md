---
name: csharp-sensor-pipeline
description: Use when building or modifying .NET 10 / C# 14 services that ingest sensor telemetry from UDP, TCP, serial/COM, Bluetooth Classic, or BLE through a five-stage pipeline (integration → protocol → normalization → alarm → final) and publish to an in-process Orchestration Manager. Triggers on mentions of Kestrel ConnectionHandler, TcpListener, System.IO.Pipelines, PipeReader, PipeWriter, ReadOnlySequence, SequenceReader, Channel<T>, bounded channels, backpressure, binary protocol decoding, sensor frame parsing, YAML-seeded SQLite sensor config, or per-pipeline context cache in a .NET service. Use whenever the user is adding a sensor type, wiring a new transport, changing stage contracts, tuning latency, or choosing between inline stages and Channel<T>-separated stages. Consult even when the user doesn't name these terms — any C# project that reads bytes from a socket or port and emits typed events is in scope.
---

# C# Sensor Pipeline (.NET 10 / C# 14)

Build sensor ingestion as a **five-stage pipeline** in-process: one instance per sensor, configured from SQLite (seeded from YAML), with an in-memory context cache per pipeline, publishing POJOs to an in-process Orchestration Manager. Transport choice lives in stage 1 and drives latency. Pipeline shape — direct method invocation vs `Channel<T>` between stages — is chosen per pipeline based on whether producer and consumer need decoupling.

This skill mirrors the Java `reactive-sensor-pipeline` skill architecturally. The stages, YAML→SQLite rule, and context-cache scoping are identical. The C# primitives (`System.IO.Pipelines`, `Channel<T>`, `PeriodicTimer`) replace Reactor Netty + Project Reactor. Security, configuration, and deployment follow the other C# skills (`csharp-windows-service`, `csharp-rest-aspnetcore`, `csharp-efcore-sqlite`) — see those for details not repeated here.

---

## Architecture at a glance

```
YAML (seed) ──► SQLite (runtime truth) ──► PipelineFactory
                                                  │
                                                  ▼
                               ┌─ PipelineInstance (per sensor) ─┐
                               │  ContextCache (Caffeine-equiv)   │
                               │  [1] Integration                 │
                               │  [2] Protocol                    │
                               │  [3] Normalization               │
                               │  [4] Alarm                       │
                               │  [5] Final ──┐                   │
                               └──────────────┼───────────────────┘
                                              ▼
                                 Channel<SensorEvent> or direct call
                                              ▼
                               ┌─ Orchestration Manager ────────┐
                               │  REST API (ASP.NET Core)       │
                               │  OAuth2 JWT Bearer             │
                               │  Subscribes to events          │
                               └────────────────────────────────┘
```

Same deployable. Same JVM — same .NET process. No network between pipeline and Orchestration Manager. The boundary is an interface, not a socket.

---

## The five stages — contract

Every pipeline flows through these stages in order. Same rules as the Java skill. If a stage wants to do another stage's job, the abstraction is wrong.

### Stage 1 — Integration
Transport-level I/O. Input: the wire. Output: `IAsyncEnumerable<ReadOnlyMemory<byte>>` or a `Channel<ReadOnlyMemory<byte>>.Reader`, one emission per logical frame.

- **Allowed:** socket/port setup, reconnect, framing (length-prefix, delimiter, timeout), backpressure strategy selection.
- **Forbidden:** decoding payload semantics, calibration, rule evaluation, cache access beyond connection state.
- **Lifecycle:** `StartAsync(CancellationToken)` and `StopAsync(CancellationToken)`. `StopAsync` releases the transport within 2s.

### Stage 2 — Protocol
Parse bytes into a typed raw reading. Input: `ReadOnlyMemory<byte>` frames. Output: `RawReading` records specific to the sensor protocol.

- **Allowed:** decoding, checksum validation, malformed-frame rejection (drop + metric, don't throw).
- **Forbidden:** unit conversion, calibration, cross-frame state beyond framing.
- **Error policy:** malformed frame increments `malformed_frames_total{sensor=...}`, logs at `Debug`, drops. Does not propagate up.

### Stage 3 — Normalization
Convert raw to canonical. Input: `RawReading`. Output: `CanonicalReading`.

- **Allowed:** unit conversion, timestamp normalization to UTC `DateTimeOffset`, coordinate transforms, per-sensor calibration from the context cache.
- **Forbidden:** threshold evaluation, any history-dependent logic.
- **Calibration:** read from cache only. Never hit SQLite on the hot path.

### Stage 4 — Alarm
Stateful rule evaluation. Input: `CanonicalReading`. Output: `SensorEvent` (reading + zero or more `AlarmAnnotation`s).

- **Allowed:** rolling windows, threshold state machines, debouncing, reads and writes to the context cache.
- **Forbidden:** I/O, blocking operations, delivery concerns.
- **State scope:** all state in the context cache. No static fields. No stage-local fields except config set once.

### Stage 5 — Final
Hand the event off to the Orchestration Manager.

- **Allowed:** method call to `ISensorEventSink.PublishAsync` or write to `Channel<SensorEvent>`, metrics, final-mile filtering.
- **Forbidden:** transforming payload, adding annotations, direct persistence.

---

## Pipeline shape — inline vs Channel<T>

### Default: direct invocation

Each stage is a class with a method; the next stage is injected; the method calls the next. Simple, synchronous-looking, fully testable per stage.

```csharp
public interface IProtocolStage
{
    ValueTask ProcessAsync(ReadOnlyMemory<byte> frame, CancellationToken ct);
}

public sealed class TemperatureProtocolStage(INormalizationStage next, ILogger<TemperatureProtocolStage> logger)
    : IProtocolStage
{
    public async ValueTask ProcessAsync(ReadOnlyMemory<byte> frame, CancellationToken ct)
    {
        if (!TryDecode(frame.Span, out var raw))
        {
            logger.LogDebug("Dropping malformed frame, length={Length}", frame.Length);
            return;
        }
        await next.ProcessAsync(raw, ct);
    }
}
```

Stages run on the caller's thread. No scheduler switch, no allocation per stage boundary, no backpressure mechanism (method calls have no backpressure — they return when done, or throw, or honor cancellation).

### Use `Channel<T>` when you have one of these:

1. **Producer and consumer run on different schedulers** — e.g., stage 1 uses a dedicated thread (serial, Bluetooth), stages 2-5 should run on the thread pool. `Channel<T>` is the handoff.
2. **Burst absorption** — the transport emits in bursts, downstream processes at steady rate. Bounded channel with `FullMode = Wait` gives natural backpressure.
3. **Fan-out** — one stage feeds multiple consumers (rare in sensor pipelines; alarm history + live event stream).
4. **Cross-pipeline aggregation** — multiple pipelines feed one consumer. Shared `Channel<SensorEvent>` is the meeting point.

When you introduce a channel, put it at **one** stage boundary — typically between stage 1 and stage 2, or between stage 5 and the Orchestration Manager. **Never a channel between every stage.** That's a bus, not a pipeline, and it destroys cache locality and stack-trace clarity.

### Rule

**Start with direct invocation. Introduce a `Channel<T>` only when one of the four conditions above is true, and introduce exactly one channel at the boundary where it's needed.**

See `references/channels.md` for bounded channel configuration, drop policies, and observability on channel depth.

---

## Transport recipes (stage 1)

Each transport has a native shape in .NET. Picking the wrong bridge adds 10-50ms of p99 latency.

| Transport | Primitive | Bridging | Scheduler |
|---|---|---|---|
| UDP | `Socket` with `ReceiveFromAsync` | None — `IAsyncEnumerable` directly | `TaskScheduler.Default` (thread pool) |
| TCP | `TcpListener` or Kestrel `ConnectionHandler` | `PipeReader` from `conn.Transport.Input` | Thread pool |
| Serial (System.IO.Ports) | Blocking `SerialPort.Read` | Dedicated thread per port, `Channel.Writer` | Dedicated thread |
| Bluetooth Classic (RFCOMM) | Blocking `NetworkStream` from BT stack | Same as serial | Dedicated thread |
| BLE (GATT notifications) | Callback-based via `Windows.Devices.Bluetooth` or `InTheHand.BluetoothLE` | `Channel.Writer` from callback | Default |

### TCP — use Kestrel's ConnectionHandler

Kestrel handles accept-loop, connection tracking, TLS termination (if needed), graceful shutdown, and exposes a `PipeReader`/`PipeWriter` on every connection. This is better than rolling `TcpListener` unless you have a reason.

```csharp
public sealed class TcpIntegrationStage(IProtocolStage next, ILogger<TcpIntegrationStage> logger)
    : ConnectionHandler
{
    public override async Task OnConnectedAsync(ConnectionContext connection)
    {
        var reader = connection.Transport.Input;
        var ct = connection.ConnectionClosed;

        while (!ct.IsCancellationRequested)
        {
            var result = await reader.ReadAsync(ct);
            var buffer = result.Buffer;

            while (TryReadFrame(ref buffer, out var frame))
            {
                await next.ProcessAsync(frame, ct);
            }

            reader.AdvanceTo(buffer.Start, buffer.End);
            if (result.IsCompleted) break;
        }
    }

    private static bool TryReadFrame(ref ReadOnlySequence<byte> buffer, out ReadOnlyMemory<byte> frame)
    {
        frame = default;
        var r = new SequenceReader<byte>(buffer);
        if (!r.TryReadBigEndian(out int length)) return false;
        if (r.Remaining < length) return false;

        var framed = buffer.Slice(r.Position, length);
        frame = framed.IsSingleSegment ? framed.First : framed.ToArray();
        buffer = buffer.Slice(framed.End);
        return true;
    }
}
```

Wire in `Program.cs`:

```csharp
builder.WebHost.ConfigureKestrel(kestrel =>
{
    kestrel.ListenAnyIP(9000, listen => listen.UseConnectionHandler<TcpIntegrationStage>());
    kestrel.ListenAnyIP(5000);  // HTTP for the Orchestration Manager coexists
});
```

This is why stage 1 uses Kestrel even when HTTP isn't otherwise involved: one server, one lifecycle, free TLS, free connection metrics, graceful shutdown wired into the host.

### Bare TcpListener alternative

Use bare `TcpListener` only when:
- The project must not reference `Microsoft.AspNetCore.App` (rare, usually size-driven).
- You're building a non-HTTP-coexisting service where adding Kestrel feels like pulling in a framework for a one-socket listener.

```csharp
public sealed class TcpIntegrationStage(IProtocolStage next) : IHostedService
{
    private TcpListener? _listener;
    private CancellationTokenSource? _cts;

    public Task StartAsync(CancellationToken _)
    {
        _cts = new CancellationTokenSource();
        _listener = new TcpListener(IPAddress.Any, 9000);
        _listener.Start();
        _ = AcceptLoopAsync(_cts.Token);
        return Task.CompletedTask;
    }

    private async Task AcceptLoopAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            var client = await _listener!.AcceptTcpClientAsync(ct);
            _ = HandleClientAsync(client, ct);
        }
    }

    private async Task HandleClientAsync(TcpClient client, CancellationToken ct)
    {
        using var _ = client;
        var pipe = PipeReader.Create(client.GetStream());
        // ... same framing loop as Kestrel version
    }

    public Task StopAsync(CancellationToken _) { _cts?.Cancel(); _listener?.Stop(); return Task.CompletedTask; }
}
```

Kestrel's ConnectionHandler is still recommended. Include this only when you've made the deliberate decision.

### UDP

```csharp
public sealed class UdpIntegrationStage(IProtocolStage next, SensorConfig config) : IHostedService
{
    private Socket? _socket;
    private CancellationTokenSource? _cts;

    public Task StartAsync(CancellationToken _)
    {
        _cts = new CancellationTokenSource();
        _socket = new Socket(AddressFamily.InterNetwork, SocketType.Dgram, ProtocolType.Udp);
        _socket.Bind(new IPEndPoint(IPAddress.Any, config.Transport.AsUdp().Port));
        _ = ReceiveLoopAsync(_cts.Token);
        return Task.CompletedTask;
    }

    private async Task ReceiveLoopAsync(CancellationToken ct)
    {
        var buffer = new byte[65_507];  // max UDP payload
        while (!ct.IsCancellationRequested)
        {
            var result = await _socket!.ReceiveFromAsync(buffer, SocketFlags.None,
                new IPEndPoint(IPAddress.Any, 0), ct);
            var frame = new ReadOnlyMemory<byte>(buffer, 0, result.ReceivedBytes);
            await next.ProcessAsync(frame, ct);
        }
    }

    public Task StopAsync(CancellationToken _) { _cts?.Cancel(); _socket?.Dispose(); return Task.CompletedTask; }
}
```

UDP is stateless. One socket serves all clients. No accept loop. Latency is the lowest of any transport — microseconds.

**Common mistake:** assuming ordered delivery. Stage 2 reorders by sequence number if the protocol needs it.

### Serial, Bluetooth Classic, BLE

See `references/transports.md` for full recipes including dedicated-thread patterns for blocking APIs, BLE GATT callback wiring, and shared-connection handling for multiplexed protocols.

---

## Configuration: YAML → SQLite → runtime

Same rule as the Java skill: **YAML is seed, SQLite is truth. SQLite wins on conflict.** Only `--reseed` overwrites.

Implementation on .NET:

- **SQLite access:** `Microsoft.Data.Sqlite` directly, or EF Core if you want the ORM. For the hot path (calibration loaded into cache at pipeline start, hot-reload refresh), direct `SqliteConnection` is simpler and faster. For admin writes from the Orchestration Manager's REST API, EF Core is fine.
- **YAML parsing:** `YamlDotNet`. Deserialize into typed records matching the SQLite schema.
- **Migrations:** if using EF Core, standard migrations (see `csharp-efcore-sqlite` skill). If raw SQLite, a simple `user_version` PRAGMA-based migrator is enough.

Schema: one row per sensor in `sensor` table, with FK'd tables for protocol config, calibration, alarm rules. Calibration and alarm rules are hot-reloadable (pushed to context cache on change). Transport and protocol changes require pipeline restart.

See `references/config.md` for schema, seeder, and hot-reload wiring.

---

## Context cache

One per pipeline instance. In-memory. Dies with the pipeline.

Use **Microsoft.Extensions.Caching.Memory** (`IMemoryCache`) or a simpler hand-rolled `ConcurrentDictionary` wrapper. `IMemoryCache` is overkill for the small, fixed key set per pipeline — prefer the hand-rolled one unless you need size eviction.

```csharp
public sealed class ContextCache
{
    private readonly ConcurrentDictionary<string, object> _store = new();

    public ContextCache(string sensorId) => SensorId = sensorId;
    public string SensorId { get; }

    public T? Get<T>(string key) where T : class
    {
        if (!_store.TryGetValue(key, out var value)) return null;
        if (value is not T typed) throw new InvalidOperationException(
            $"key={key} expected={typeof(T).Name} actual={value.GetType().Name}");
        return typed;
    }

    public T GetRequired<T>(string key) where T : class
        => Get<T>(key) ?? throw new KeyNotFoundException(key);

    public void Put(string key, object value) => _store[key] = value;

    public T GetOrAdd<T>(string key, Func<T> factory) where T : class
        => (T)_store.GetOrAdd(key, _ => factory());
}
```

Keying convention: `<category>.<name>` — `calibration.offset`, `alarm.high_temp_state`, `stream.last_seq`. Enforced by code review.

Same scoping rules as the Java skill:

- **In the cache:** calibration, alarm state machines, stream position/gap detection.
- **Not in the cache:** transport state (lives in stage 1 fields), persistent state (SQLite), cross-pipeline state (doesn't exist here).

For concurrent read-modify-write from stage 4, use `Interlocked` on atomic types stored in the cache, or wrap values in `AtomicReference`-equivalent:

```csharp
var counter = cache.GetOrAdd("alarm.high_count", () => new StrongBox<long>(0));
Interlocked.Increment(ref counter.Value);
```

See `references/context-cache.md` for lifecycle details and hot-reload patterns.

---

## Pipeline construction

A `PipelineFactory` reads one sensor row from SQLite, resolves stage implementations per sensor type from DI, wires them together, and returns a `PipelineInstance` with `Start`/`Stop`. Same pattern as the Java skill's factory.

```csharp
public sealed class PipelineFactory(
    IEnumerable<IIntegrationStageFactory> integrationFactories,
    IProtocolStageRegistry protocols,
    INormalizationStageRegistry normalizers,
    IAlarmStageRegistry alarms,
    ISensorEventSink finalSink,
    ISensorConfigRepository repo,
    ILoggerFactory loggerFactory)
{
    public PipelineInstance Build(string sensorId)
    {
        var config = repo.LoadEnabled(sensorId)
            ?? throw new InvalidOperationException($"no config: {sensorId}");

        var cache = new ContextCache(sensorId);
        LoadCalibrationInto(cache, config);
        LoadAlarmRulesInto(cache, config);

        var finalStage = new FinalStage(finalSink, config,
            loggerFactory.CreateLogger<FinalStage>());
        var alarmStage = alarms.Resolve(config.SensorType, cache, finalStage);
        var normStage  = normalizers.Resolve(config.SensorType, cache, alarmStage);
        var protoStage = protocols.Resolve(config.Protocol, normStage);

        var integrationFactory = integrationFactories
            .FirstOrDefault(f => f.Supports(config.Transport))
            ?? throw new InvalidOperationException(
                $"no integration for transport: {config.Transport.Type}");

        var integration = integrationFactory.Create(config, protoStage);

        return new PipelineInstance(sensorId, integration, cache);
    }
}
```

Stages are wired **from the end backwards** (final → alarm → normalization → protocol → integration) because each stage constructs with a reference to the next. The final stage has no "next" — it writes to the sink.

**Registry pattern, same as Java skill**: add a new protocol by implementing `IProtocolStageFactory` + `[Component]` equivalent (i.e., register it in DI). Don't edit `PipelineFactory`.

For the full factory, `PipelineInstance` lifecycle, and `PipelineManager` (owns all instances, reacts to config changes), see `references/pipeline-construction.md`.

---

## Final stage and the Orchestration Manager boundary

Two delivery modes:

### Direct call (default)

```csharp
public interface ISensorEventSink
{
    ValueTask PublishAsync(SensorEvent evt, CancellationToken ct);
}
```

Pipeline calls `sink.PublishAsync`. Orchestration Manager implements `ISensorEventSink`, decides what to do with the event (persist, push to subscribers, broadcast).

This is an in-process method call. No serialization, no queue, no network. Latency is measured in microseconds.

### Channel-based (when decoupling is needed)

```csharp
public sealed class ChannelSensorEventSink : ISensorEventSink
{
    private readonly Channel<SensorEvent> _channel;

    public ChannelSensorEventSink()
    {
        _channel = Channel.CreateBounded<SensorEvent>(new BoundedChannelOptions(1024)
        {
            FullMode = BoundedChannelFullMode.DropOldest,
            SingleReader = false,
            SingleWriter = false,
        });
    }

    public ChannelReader<SensorEvent> Reader => _channel.Reader;

    public ValueTask PublishAsync(SensorEvent evt, CancellationToken ct)
    {
        if (!_channel.Writer.TryWrite(evt))
        {
            // metric: drop
        }
        return ValueTask.CompletedTask;
    }
}
```

The Orchestration Manager subscribes to `Reader` from a background task. This is the right shape when:

- The Orchestration Manager does expensive work per event (DB writes, HTTP fan-out).
- You want to absorb bursts without blocking the pipeline.
- Drop policy matters — `DropOldest` preserves recency over completeness, which is usually right for sensors.

**`TryWrite`, not `WriteAsync`.** `WriteAsync` would await on a full channel, pushing backpressure back through the pipeline — which for sensor data is wrong. Losing old events is better than blocking ingress.

### Which to use

Start with direct call. Switch to channel when profiling shows the Orchestration Manager's work is pushing latency into the pipeline. Don't speculatively decouple.

---

## Latency budgets

Per-sensor, not global. Declared in YAML as `p99_budget_ms`. The pipeline measures end-to-end from stage 1 frame receipt to `ISensorEventSink.PublishAsync` return, emits `sensor_latency_ms{sensor=...}` histogram via `System.Diagnostics.Metrics`.

Budget violations are metric signals, not exceptions. Alert on the metric; don't throw.

### Allocation discipline on the hot path

- **Avoid `ToArray()` on `ReadOnlySequence<byte>`.** Use `IsSingleSegment` + `First` where possible; fall back to rent-and-copy via `ArrayPool<byte>.Shared`.
- **`stackalloc` for small fixed-size decode buffers.** `Span<byte> buf = stackalloc byte[8]` for decoding a `double`.
- **`record struct` instead of `record class`** for readings on the hot path. Stack-allocated, no GC pressure. `readonly record struct` for stronger immutability.
- **`ValueTask<T>` instead of `Task<T>`** for stage methods that usually complete synchronously (validation drops, cache hits).
- **`IMemoryCache` / `Caffeine` allocate per eviction event.** The hand-rolled `ConcurrentDictionary` cache above doesn't.
- **Avoid LINQ on the hot path.** A `for` loop over a `List<T>` is zero-allocation; `source.Where(...).ToList()` allocates an iterator and the result list. For stages 3-4 processing one reading at a time, this is moot — but stage 2 parsing batched telemetry can suffer.

---

## Security

Same as the Java skill: two surfaces, only one matters here.

**Pipeline runner → Orchestration Manager:** no auth. Same process. See the REST-vs-pipeline boundary in `csharp-rest-aspnetcore`'s auth reference — it's an in-process method call, auth is nonsensical.

**Client → Orchestration Manager REST:** ASP.NET Core with `AddAuthentication().AddJwtBearer()`, see `csharp-rest-aspnetcore` for the full setup.

**Input validation in the pipeline:**
- Stage 2: malformed frame → metric, drop.
- Stage 3: out-of-physical-range values (NaN, Infinity, -300°C temperature) → metric, drop.
- Stage 4: business rules, not validation.

---

## Testing

Same three-tier structure as the Java skill, with C# primitives:

- **Stage unit tests** — each stage implements an interface; mock the "next" stage; feed synthetic input; assert.
- **Transport tests** — fake transport (in-memory `Channel<ReadOnlyMemory<byte>>`) drives stage 1. Real transports tested against loopback.
- **Pipeline tests** — wire five real stages, fake transport, in-memory sink; assert events landing.
- **Config tests** — in-memory SQLite + YAML seeder; assert SQLite-wins rule.

```csharp
public class TemperatureProtocolStageTests
{
    [Fact]
    public async Task ParsesValidFrame()
    {
        var next = Substitute.For<INormalizationStage>();
        var stage = new TemperatureProtocolStage(next,
            NullLogger<TemperatureProtocolStage>.Instance);

        var frame = Convert.FromHexString("aa0123047f00");
        await stage.ProcessAsync(frame, CancellationToken.None);

        await next.Received(1).ProcessAsync(
            Arg.Is<RawTemperatureReading>(r => r.SensorId == 0x0123),
            Arg.Any<CancellationToken>());
    }

    [Fact]
    public async Task DropsFrameWithBadChecksum()
    {
        var next = Substitute.For<INormalizationStage>();
        var stage = new TemperatureProtocolStage(next,
            NullLogger<TemperatureProtocolStage>.Instance);

        await stage.ProcessAsync(Convert.FromHexString("aa0123047f01"), default);

        await next.DidNotReceive().ProcessAsync(Arg.Any<RawTemperatureReading>(), default);
    }
}
```

No test touches real hardware. See `references/testing.md` for fake-transport helpers, channel-based pipeline tests, and deterministic timing with `TimeProvider`.

---

## C# 14 idioms worth using here

- **`readonly record struct`** for hot-path readings — stack-allocated, value-equality, immutable.
- **Primary constructors** for stages (`TemperatureProtocolStage(INormalizationStage next, ILogger logger)`).
- **`required` members** on `SensorConfig` and config records.
- **Collection expressions** (`[]`) for default empty annotation lists on events.
- **Pattern matching on `Transport` configs** via sealed type hierarchies:

```csharp
public abstract record TransportConfig;
public sealed record UdpTransport(int Port) : TransportConfig;
public sealed record TcpTransport(int Port) : TransportConfig;
public sealed record SerialTransport(string PortName, int BaudRate) : TransportConfig;

var message = config switch
{
    UdpTransport udp => $"UDP on {udp.Port}",
    TcpTransport tcp => $"TCP on {tcp.Port}",
    SerialTransport ser => $"Serial on {ser.PortName}@{ser.BaudRate}",
    _ => throw new UnreachableException(),
};
```

**What to avoid on the hot path:**
- `IEnumerable<T>.Where(...).Select(...).ToList()` — allocates iterators and lists.
- `string.Format` / interpolation that allocates — use `Utf8Formatter` or `IBufferWriter<T>` for hot logging.
- Boxing from generic methods on value types — watch for `object` constraints.

---

## What this skill does not cover

- **HTTP-only services.** Use `csharp-rest-aspnetcore`.
- **Background jobs without transport.** Use `csharp-windows-service`'s `BackgroundService` patterns.
- **Database access beyond SQLite config reads.** Use `csharp-efcore-sqlite`.
- **Non-sensor binary protocols.** The skill is tuned for sensor telemetry characteristics (small frames, high frequency, per-sensor latency budgets). For large binary payloads (file transfer, image streaming), different tradeoffs apply.
- **Distributed sensor networks.** If pipelines run on different machines, you need a real message broker (Kafka, RabbitMQ, NATS) — out of scope here.
