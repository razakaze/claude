# Channels

`System.Threading.Channels` is the .NET answer to Reactor's `Sinks.Many` — producer-consumer with bounded or unbounded capacity, multi or single reader/writer, configurable overflow policy. Cheap, fast, correct.

## When to introduce a channel

Only when one of these is true:

1. **Producer and consumer run on different schedulers.** Dedicated thread (serial, Bluetooth) → async pipeline. Channel is the sync/async boundary.
2. **Burst absorption.** Input is spiky, downstream steady. Bounded channel with `FullMode = Wait` gives backpressure.
3. **Fan-out.** One producer, many consumers.
4. **Cross-pipeline aggregation.** Multiple pipelines produce to one shared consumer (the Orchestration Manager).

Not when "it might be useful later." Channels add a queue between two components, which adds latency (context switches, cache misses on dequeue), adds an observable queue depth you have to monitor, and adds a failure mode (channel full).

## Bounded vs unbounded

**Always bounded.** An unbounded channel is a memory leak waiting to happen. Decide the capacity and the full-policy explicitly.

```csharp
var channel = Channel.CreateBounded<SensorEvent>(new BoundedChannelOptions(1024)
{
    FullMode = BoundedChannelFullMode.DropOldest,
    SingleReader = true,
    SingleWriter = false,
});
```

## FullMode choices

| Mode | Semantics | When |
|---|---|---|
| `Wait` | Writer awaits until space. Backpressure propagates. | The producer is pressure-tolerant and you want lossless delivery. |
| `DropOldest` | Channel drops oldest entry to make room. Latest prevails. | Sensor telemetry — freshness beats completeness. |
| `DropNewest` | Drop new entry instead of admitting. | Rare. When old state is sacred. |
| `DropWrite` | Reject write (`TryWrite` returns false). Caller decides. | When the caller wants a signal to handle explicitly. |

**Default for sensor pipelines: `DropOldest`.** Losing yesterday's reading because today's flood is too large is fine.

## SingleReader / SingleWriter

Set true when you know the topology. Enables fast paths in the channel implementation. Default is false for both.

```csharp
// Stage 1 reader thread writes, stage 2 loop reads: SR, SW both true.
new BoundedChannelOptions(256)
{
    SingleReader = true,
    SingleWriter = true,
    FullMode = BoundedChannelFullMode.DropOldest,
}
```

Mis-configuring these (claiming single writer when there are many) causes subtle corruption. If in doubt, leave as false.

## TryWrite vs WriteAsync

**`TryWrite`** — synchronous, returns `bool`. Fails if channel is full (with `FullMode.Wait`) or completed.

**`WriteAsync`** — awaits if channel is full. Use only when you want backpressure propagation.

For the final-stage sink emitting to the Orchestration Manager: `TryWrite`, increment a drop metric on `false`. Don't block the pipeline because downstream is slow.

For the reader-thread → pipeline boundary in the serial stage: `TryWrite` with `DropOldest`. If the consumer can't keep up with serial data, the right behavior is to drop, not to stall the reader.

## ReadAllAsync

The idiomatic consumer:

```csharp
await foreach (var frame in channel.Reader.ReadAllAsync(ct))
{
    await ProcessAsync(frame, ct);
}
```

Completes when the channel is completed (`Writer.Complete()` or `Writer.TryComplete()`) and empty, or when `ct` fires.

## Completing the channel

Signal the consumer that no more items are coming:

```csharp
_channel.Writer.TryComplete();
```

After complete, `TryWrite` returns false, `WriteAsync` throws. Consumer loops drain remaining items, then exit cleanly.

**Always complete the channel on shutdown.** Leaked consumer loops waiting on a never-completed channel are a classic lifetime bug.

## Channel between stages — a worked example

Use case: stage 1 is serial (dedicated thread), stages 2-5 run on the thread pool.

```csharp
public sealed class SerialToProtocolBridge(
    IProtocolStage next,
    ILogger<SerialToProtocolBridge> logger) : IIntegrationDownstream
{
    private readonly Channel<byte[]> _channel = Channel.CreateBounded<byte[]>(
        new BoundedChannelOptions(256)
        {
            FullMode = BoundedChannelFullMode.DropOldest,
            SingleReader = true,
            SingleWriter = true,
        });

    public Task StartConsumerAsync(CancellationToken ct) =>
        Task.Run(async () =>
        {
            await foreach (var frame in _channel.Reader.ReadAllAsync(ct))
            {
                try
                {
                    await next.ProcessAsync(frame, ct);
                }
                catch (Exception ex)
                {
                    logger.LogWarning(ex, "Protocol stage threw, dropping frame");
                }
            }
        }, ct);

    public void Publish(byte[] frame)
    {
        if (!_channel.Writer.TryWrite(frame))
        {
            // Metric: drop
        }
    }

    public void Complete() => _channel.Writer.TryComplete();
}
```

The reader thread calls `Publish`. The consumer loop runs on the thread pool via `Task.Run`. Channel capacity and overflow are explicit. Completion is explicit.

## Observability

Channel depth is a leading indicator of backpressure. Expose it:

```csharp
_ = Task.Run(async () =>
{
    var meter = new Meter("MyCompany.SensorPipeline");
    meter.CreateObservableGauge("channel_depth", () => _channel.Reader.Count,
        description: "Current channel depth");
}, ct);
```

Alert on sustained high depth. A channel at 90% capacity means you're one burst away from dropping frames. Either scale the consumer or accept the drops.

## Anti-patterns

**Many channels for one pipeline.** Channel between every stage is not a pipeline, it's a bus. Method calls between stages are nearly free; channels between them add queuing latency at every boundary. One or two channels per pipeline at principled boundaries, not five.

**Unbounded channel.** Memory grows without bound. Every unbounded channel is a pending OOM crash.

**Channel as a replacement for a method call.** If two components run on the same scheduler with direct relationship, call the method. A channel with `SingleReader` and `SingleWriter` and capacity 1 and `FullMode.Wait` is a bad re-implementation of `await`.

**Not completing the channel.** Producer exits, consumer loops forever on `ReadAllAsync`. The shutdown path never completes. Always `TryComplete` when done producing.
