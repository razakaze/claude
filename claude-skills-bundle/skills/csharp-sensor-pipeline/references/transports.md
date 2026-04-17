# Transport Recipes

One section per transport. Minimal code, bridging decision, latency, common mistakes.

---

## UDP

Stateless, one socket serves all peers. Lowest latency of any transport.

```csharp
public sealed class UdpIntegrationStage(
    IProtocolStage next,
    SensorConfig config,
    ILogger<UdpIntegrationStage> logger) : IHostedService
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
        var buffer = ArrayPool<byte>.Shared.Rent(65_507);
        try
        {
            while (!ct.IsCancellationRequested)
            {
                SocketReceiveFromResult result;
                try
                {
                    result = await _socket!.ReceiveFromAsync(buffer, SocketFlags.None,
                        new IPEndPoint(IPAddress.Any, 0), ct);
                }
                catch (OperationCanceledException) { break; }
                catch (SocketException ex)
                {
                    logger.LogWarning(ex, "UDP receive failed, continuing");
                    continue;
                }

                var frame = new ReadOnlyMemory<byte>(buffer, 0, result.ReceivedBytes);
                await next.ProcessAsync(frame, ct);
            }
        }
        finally
        {
            ArrayPool<byte>.Shared.Return(buffer);
        }
    }

    public Task StopAsync(CancellationToken _)
    {
        _cts?.Cancel();
        _socket?.Dispose();
        return Task.CompletedTask;
    }
}
```

**Latency:** microseconds. Single dedicated receive loop.
**Common mistake:** reusing the buffer across emissions. `next.ProcessAsync` might outlive the receive call if stage 2 queues the frame. Copy before queueing if so — for direct-invocation pipelines, the call has returned before the next receive, so the shared buffer is fine.
**Common mistake:** assuming ordered delivery. If the protocol needs order, stage 2 reorders by sequence number.

---

## TCP via Kestrel ConnectionHandler

Preferred when you want Kestrel's accept loop, connection tracking, optional TLS, and graceful shutdown for free.

```csharp
public sealed class TcpIntegrationStage(
    IProtocolStage next,
    ILogger<TcpIntegrationStage> logger) : ConnectionHandler
{
    public override async Task OnConnectedAsync(ConnectionContext connection)
    {
        var reader = connection.Transport.Input;
        var ct = connection.ConnectionClosed;

        try
        {
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
        catch (OperationCanceledException) { }
        catch (ConnectionResetException) { }
        catch (Exception ex)
        {
            logger.LogWarning(ex, "Connection error on {Id}", connection.ConnectionId);
        }
    }

    private static bool TryReadFrame(ref ReadOnlySequence<byte> buffer, out ReadOnlyMemory<byte> frame)
    {
        frame = default;
        var reader = new SequenceReader<byte>(buffer);
        if (!reader.TryReadBigEndian(out int length)) return false;
        if (length <= 0 || length > 64_000) { buffer = buffer.Slice(buffer.End); return false; }
        if (reader.Remaining < length) return false;

        var payload = buffer.Slice(reader.Position, length);
        frame = payload.IsSingleSegment
            ? payload.First
            : payload.ToArray();  // span across segments, copy
        buffer = buffer.Slice(payload.End);
        return true;
    }
}
```

Wire in `Program.cs`:

```csharp
builder.WebHost.ConfigureKestrel(k =>
{
    k.ListenAnyIP(9000, listen => listen.UseConnectionHandler<TcpIntegrationStage>());
    k.ListenAnyIP(5000);  // HTTP for Orchestration Manager
});
```

Register the handler in DI:

```csharp
builder.Services.AddSingleton<TcpIntegrationStage>();
```

**Latency:** microseconds plus TCP ack round-trips.
**Framing:** length-prefix shown. For delimiter framing (e.g., `\n`), use `SequenceReader.TryReadTo(out ReadOnlySequence<byte> seq, (byte)'\n')`.
**Common mistake:** calling `buffer.ToArray()` unconditionally. For the common case where the frame is in one segment, `buffer.First` avoids the copy.
**Common mistake:** not advancing past invalid/oversized frames. If `length` is wrong the buffer stays stuck; advance past it and let the protocol resync on the next valid frame.

## TCP via bare TcpListener

Only when Kestrel is not wanted:

```csharp
public sealed class BareTcpIntegrationStage(
    IProtocolStage next,
    SensorConfig config) : IHostedService
{
    private TcpListener? _listener;
    private CancellationTokenSource? _cts;

    public Task StartAsync(CancellationToken _)
    {
        _cts = new CancellationTokenSource();
        _listener = new TcpListener(IPAddress.Any, config.Transport.AsTcp().Port);
        _listener.Start();
        _ = AcceptLoopAsync(_cts.Token);
        return Task.CompletedTask;
    }

    private async Task AcceptLoopAsync(CancellationToken ct)
    {
        while (!ct.IsCancellationRequested)
        {
            TcpClient client;
            try { client = await _listener!.AcceptTcpClientAsync(ct); }
            catch (OperationCanceledException) { break; }
            _ = Task.Run(() => HandleClientAsync(client, ct), ct);
        }
    }

    private async Task HandleClientAsync(TcpClient client, CancellationToken ct)
    {
        using var _ = client;
        var pipe = PipeReader.Create(client.GetStream());
        // ... same framing loop as the Kestrel handler's OnConnectedAsync
    }

    public Task StopAsync(CancellationToken _)
    {
        _cts?.Cancel();
        _listener?.Stop();
        return Task.CompletedTask;
    }
}
```

## Serial (System.IO.Ports)

Blocking API. Dedicated thread per port. `Channel<T>` bridges to the pipeline.

```csharp
public sealed class SerialIntegrationStage(
    IProtocolStage next,
    SensorConfig config,
    ILogger<SerialIntegrationStage> logger) : IHostedService
{
    private SerialPort? _port;
    private Thread? _readerThread;
    private CancellationTokenSource? _cts;
    private readonly Channel<byte[]> _channel = Channel.CreateBounded<byte[]>(
        new BoundedChannelOptions(256) { FullMode = BoundedChannelFullMode.DropOldest });

    public async Task StartAsync(CancellationToken ct)
    {
        var cfg = config.Transport.AsSerial();
        _port = new SerialPort(cfg.PortName, cfg.BaudRate, cfg.Parity, cfg.DataBits, cfg.StopBits)
        {
            ReadTimeout = 100,
        };
        _port.Open();

        _cts = new CancellationTokenSource();
        _readerThread = new Thread(() => ReadLoop(_cts.Token))
        {
            IsBackground = true,
            Name = $"serial-{config.SensorId}",
        };
        _readerThread.Start();

        _ = ConsumeLoopAsync(_cts.Token);
    }

    private void ReadLoop(CancellationToken ct)
    {
        var buf = new byte[512];
        while (!ct.IsCancellationRequested && _port!.IsOpen)
        {
            try
            {
                int n = _port.Read(buf, 0, buf.Length);
                if (n > 0)
                {
                    var copy = new byte[n];
                    Buffer.BlockCopy(buf, 0, copy, 0, n);
                    _channel.Writer.TryWrite(copy);
                }
            }
            catch (TimeoutException) { /* expected, loop */ }
            catch (OperationCanceledException) { break; }
            catch (Exception ex)
            {
                logger.LogWarning(ex, "Serial read failed");
            }
        }
    }

    private async Task ConsumeLoopAsync(CancellationToken ct)
    {
        await foreach (var frame in _channel.Reader.ReadAllAsync(ct))
        {
            await next.ProcessAsync(frame, ct);
        }
    }

    public Task StopAsync(CancellationToken _)
    {
        _cts?.Cancel();
        _port?.Close();
        _channel.Writer.TryComplete();
        _readerThread?.Join(TimeSpan.FromSeconds(2));
        return Task.CompletedTask;
    }
}
```

**Why a dedicated thread, not `Task.Run`:** the Task Pool's scheduling jitter is fine for 10ms budgets but destroys tighter p99 targets. A dedicated thread is a fixed scheduling context.

**Why a Channel between reader thread and pipeline:** the reader thread is blocking and synchronous. The rest of the pipeline is async. The channel is the async/sync boundary.

**Common mistake:** no-op on `TimeoutException`. The timeout is intentional — it lets the loop check cancellation. Catch it and continue.
**Common mistake:** sharing the buffer across emissions. Always copy (`Buffer.BlockCopy` or `byte[].CopyTo`).

## Bluetooth Classic (RFCOMM)

Structurally identical to serial. Different connection API (pair + open RFCOMM socket), same read-loop shape. On Windows, `32feet.NET` (`InTheHand.Net.Bluetooth`) provides `BluetoothClient`:

```csharp
var client = new BluetoothClient();
client.Connect(new BluetoothAddress(macBytes), BluetoothService.SerialPort);
var stream = client.GetStream();
// ... same blocking read loop as serial, bytes from stream.Read
```

Pairing is a system-level concern. Do it out of band. Assume paired when the pipeline starts. Don't try to pair inside stage 1.

**Latency:** BT Classic has ~30-50ms inherent. Code can't beat that.

## BLE (GATT notifications)

Callback-based. Wrap with `Channel<T>` from the callback.

On Windows (UWP APIs via `Windows.Devices.Bluetooth`):

```csharp
public sealed class BleIntegrationStage(
    IProtocolStage next,
    SensorConfig config) : IHostedService
{
    private readonly Channel<byte[]> _channel = Channel.CreateBounded<byte[]>(
        new BoundedChannelOptions(128) { FullMode = BoundedChannelFullMode.DropOldest });
    private GattCharacteristic? _characteristic;
    private CancellationTokenSource? _cts;

    public async Task StartAsync(CancellationToken ct)
    {
        var cfg = config.Transport.AsBle();
        var device = await BluetoothLEDevice.FromBluetoothAddressAsync(ulong.Parse(cfg.Mac, System.Globalization.NumberStyles.HexNumber));
        var services = await device.GetGattServicesForUuidAsync(cfg.ServiceUuid);
        var chars = await services.Services[0].GetCharacteristicsForUuidAsync(cfg.CharacteristicUuid);
        _characteristic = chars.Characteristics[0];

        _characteristic.ValueChanged += OnValueChanged;
        await _characteristic.WriteClientCharacteristicConfigurationDescriptorAsync(
            GattClientCharacteristicConfigurationDescriptorValue.Notify);

        _cts = new CancellationTokenSource();
        _ = ConsumeLoopAsync(_cts.Token);
    }

    private void OnValueChanged(GattCharacteristic sender, GattValueChangedEventArgs args)
    {
        var data = new byte[args.CharacteristicValue.Length];
        Windows.Storage.Streams.DataReader.FromBuffer(args.CharacteristicValue).ReadBytes(data);
        _channel.Writer.TryWrite(data);
    }

    private async Task ConsumeLoopAsync(CancellationToken ct)
    {
        await foreach (var frame in _channel.Reader.ReadAllAsync(ct))
        {
            await next.ProcessAsync(frame, ct);
        }
    }

    public Task StopAsync(CancellationToken _)
    {
        if (_characteristic is not null) _characteristic.ValueChanged -= OnValueChanged;
        _cts?.Cancel();
        _channel.Writer.TryComplete();
        return Task.CompletedTask;
    }
}
```

For cross-platform BLE, use `Plugin.BLE` (MAUI-friendly) or `InTheHand.BluetoothLE`. The wrapping pattern is the same.

**Latency:** dominated by BLE connection interval (7.5-30ms). Code can't improve it.
**OverflowStrategy:** `DropOldest` for sensors where only the most recent reading matters. `Wait` (`FullMode.Wait`) only if you genuinely need every reading and can accept backpressure.
**Common mistake:** forgetting to write the CCCD descriptor (`WriteClientCharacteristicConfigurationDescriptorAsync`). Without it, the device doesn't send notifications — the pipeline looks dead.
