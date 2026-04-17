# Transport Recipes

One section per transport. Each section: minimal code, the bridging decision, latency characteristics, common mistakes.

---

## UDP

Use `reactor.netty.udp.UdpServer`. Datagrams are naturally frame-aligned — one `DatagramPacket` is one frame. No framing logic needed.

```java
public final class UdpIntegrationStage implements IntegrationStage {

    public Flux<byte[]> start(SensorConfig config) {
        UdpConfig udp = config.transport().asUdp();
        return UdpServer.create()
            .host(udp.host())
            .port(udp.port())
            .handle((in, out) -> in.receive()
                .asByteArray()
                .doOnNext(bytes -> /* metric */))
            .bind()
            .flatMapMany(conn -> /* expose as Flux via Sinks or inbound receive */);
    }
}
```

**Latency:** microseconds in the event loop. Don't bridge to another scheduler.
**Common mistake:** assuming ordered delivery. UDP doesn't guarantee it — if the sensor protocol needs ordering, stage 2 reorders by sequence number.

---

## TCP

Use `reactor.netty.tcp.TcpServer` (if the sensor connects to you) or `TcpClient` (if you connect to the sensor).

```java
public final class TcpIntegrationStage implements IntegrationStage {

    public Flux<byte[]> start(SensorConfig config) {
        TcpConfig tcp = config.transport().asTcp();
        return TcpClient.create()
            .host(tcp.host())
            .port(tcp.port())
            .connect()
            .flatMapMany(conn -> conn.inbound().receive().asByteArray());
    }
}
```

**Framing:** TCP is a byte stream, not a frame stream. Add a `LengthFieldBasedFrameDecoder` or a delimiter-based handler to the Netty pipeline before the `Flux` hand-off, or implement framing in stage 2 if the protocol is simple.

**Latency:** microseconds. No bridging.
**Common mistake:** doing framing in stage 2 when the sensor protocol has a well-defined length prefix. Use Netty's decoders — they're tested and they operate on `ByteBuf` without extra copies.

---

## Serial (jSerialComm)

Blocking API. Dedicated single thread per port. Bridge via `Sinks.Many`.

```java
public final class SerialIntegrationStage implements IntegrationStage {

    private final Sinks.Many<byte[]> sink = Sinks.many().unicast().onBackpressureBuffer();
    private SerialPort port;
    private Thread reader;

    public Flux<byte[]> start(SensorConfig config) {
        SerialConfig cfg = config.transport().asSerial();
        port = SerialPort.getCommPort(cfg.portName());
        port.setComPortParameters(cfg.baud(), cfg.dataBits(), cfg.stopBits(), cfg.parity());
        port.setComPortTimeouts(SerialPort.TIMEOUT_READ_SEMI_BLOCKING, 100, 0);
        if (!port.openPort()) throw new IllegalStateException("open failed: " + cfg.portName());

        reader = new Thread(this::readLoop, "serial-" + config.sensorId());
        reader.setDaemon(true);
        reader.start();
        return sink.asFlux();
    }

    private void readLoop() {
        byte[] buf = new byte[512];
        while (!Thread.currentThread().isInterrupted() && port.isOpen()) {
            int n = port.readBytes(buf, buf.length);
            if (n > 0) sink.tryEmitNext(java.util.Arrays.copyOf(buf, n));
        }
    }

    public void stop() {
        reader.interrupt();
        port.closePort();
        sink.tryEmitComplete();
    }
}
```

**Latency:** bounded by the read timeout (100ms above — tune per sensor). The single dedicated thread avoids `boundedElastic` scheduling jitter.
**Common mistake:** sharing a thread across ports, or using `boundedElastic`. Both destroy p99.
**Common mistake:** not calling `Arrays.copyOf` — emitting the shared `buf` causes downstream corruption.

---

## Bluetooth Classic (RFCOMM)

Same shape as serial. The connection API differs (pair, open RFCOMM socket), but once you have the `InputStream`, the read loop is identical.

On Linux, use BlueZ via `bluez-dbus`. On Windows, use `bluecove` or a JNI wrapper. Isolate the platform difference behind an `RfcommConnection` interface — stage 1 depends on the interface, not the platform library.

**Latency:** BT Classic has ~30-50ms inherent latency. Don't expect to beat that regardless of code quality.
**Common mistake:** trying to pair inside the pipeline. Pairing is a system-level concern — do it out of band, assume the device is paired when the pipeline starts.

---

## BLE (GATT notifications)

Event-driven already. Wrap the notification callback with `Flux.create`.

```java
public final class BleIntegrationStage implements IntegrationStage {

    public Flux<byte[]> start(SensorConfig config) {
        BleConfig cfg = config.transport().asBle();
        return Flux.create(sink -> {
            BluetoothDevice device = adapter.getRemoteDevice(cfg.mac());
            BluetoothGatt gatt = device.connectGatt(ctx, false, new BluetoothGattCallback() {
                @Override public void onCharacteristicChanged(BluetoothGatt g,
                                                              BluetoothGattCharacteristic c) {
                    sink.next(c.getValue());
                }
                @Override public void onConnectionStateChange(BluetoothGatt g, int status, int state) {
                    if (state == BluetoothProfile.STATE_DISCONNECTED) sink.complete();
                }
            });
            sink.onDispose(() -> { gatt.disconnect(); gatt.close(); });
        }, FluxSink.OverflowStrategy.LATEST);
    }
}
```

**Latency:** connection interval dominates — typically 7.5-30ms per notification depending on negotiated interval. Code can't improve this.
**OverflowStrategy:** `LATEST` for sensors where only the most recent reading matters (IMU, temperature). `BUFFER` with a bounded size for sensors where every reading matters (heart rate R-R intervals). Never `ERROR`.
**Common mistake:** forgetting to enable notifications on the GATT characteristic (write the CCCD descriptor). No bytes will arrive and the pipeline will look dead.
