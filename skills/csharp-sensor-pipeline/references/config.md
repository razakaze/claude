# Configuration: YAML → SQLite → Runtime

## The rule

YAML is seed. SQLite is runtime truth. **SQLite wins on conflict.** The only way YAML overwrites SQLite is an explicit `--reseed` flag.

## Startup flow

```
1. Open SQLite, run migrations (or EnsureCreated for dev).
2. For each sensor in YAML:
     - If sensor_id exists in SQLite → do nothing.
     - If not → insert from YAML.
3. Load all enabled sensors from SQLite.
4. PipelineManager.StartAll()
```

## Schema

Same shape as the Java skill. SQLite is identical regardless of language.

```sql
CREATE TABLE sensor (
    sensor_id       TEXT PRIMARY KEY,
    sensor_type     TEXT NOT NULL,
    enabled         INTEGER NOT NULL DEFAULT 1,
    transport       TEXT NOT NULL,              -- JSON
    p99_budget_ms   INTEGER NOT NULL,
    updated_at      TEXT NOT NULL
);

CREATE TABLE sensor_protocol_config (
    sensor_id TEXT PRIMARY KEY REFERENCES sensor(sensor_id) ON DELETE CASCADE,
    protocol  TEXT NOT NULL,
    params    TEXT NOT NULL                     -- JSON
);

CREATE TABLE sensor_calibration (
    sensor_id  TEXT PRIMARY KEY REFERENCES sensor(sensor_id) ON DELETE CASCADE,
    params     TEXT NOT NULL,                   -- JSON
    updated_at TEXT NOT NULL
);

CREATE TABLE sensor_alarm_rule (
    rule_id   INTEGER PRIMARY KEY AUTOINCREMENT,
    sensor_id TEXT NOT NULL REFERENCES sensor(sensor_id) ON DELETE CASCADE,
    rule_type TEXT NOT NULL,
    params    TEXT NOT NULL,                    -- JSON
    enabled   INTEGER NOT NULL DEFAULT 1
);

CREATE INDEX idx_sensor_alarm_rule_sensor_id ON sensor_alarm_rule(sensor_id);
```

JSON columns for variable-shape data (transport config varies per transport type; protocol params vary per protocol).

## Access pattern

- **Startup reads:** `Microsoft.Data.Sqlite` directly. Blocking is fine — it's once per sensor, at startup.
- **Runtime reads from pipeline stages:** none. Stages read from context cache only.
- **Writes (admin endpoint on Orchestration Manager):** EF Core if present, otherwise `Microsoft.Data.Sqlite`. Writes are rare.

## Seeder

```csharp
public sealed class YamlSensorSeeder(ILogger<YamlSensorSeeder> logger)
{
    public int Seed(string yamlPath, string connectionString, bool reseed)
    {
        var deserializer = new YamlDotNet.Serialization.DeserializerBuilder()
            .WithNamingConvention(YamlDotNet.Serialization.NamingConventions.UnderscoredNamingConvention.Instance)
            .Build();

        var yaml = deserializer.Deserialize<SensorSeedFile>(File.ReadAllText(yamlPath));

        using var conn = new SqliteConnection(connectionString);
        conn.Open();

        int inserted = 0;
        foreach (var sensor in yaml.Sensors)
        {
            if (!reseed && Exists(conn, sensor.SensorId))
            {
                logger.LogDebug("Sensor {Id} already in SQLite, skipping", sensor.SensorId);
                continue;
            }
            UpsertSensor(conn, sensor);
            inserted++;
        }

        return inserted;
    }

    private static bool Exists(SqliteConnection conn, string sensorId)
    {
        using var cmd = conn.CreateCommand();
        cmd.CommandText = "SELECT 1 FROM sensor WHERE sensor_id = $id LIMIT 1";
        cmd.Parameters.AddWithValue("$id", sensorId);
        return cmd.ExecuteScalar() is not null;
    }

    private static void UpsertSensor(SqliteConnection conn, SensorSeed s)
    {
        using var tx = conn.BeginTransaction();
        using (var cmd = conn.CreateCommand())
        {
            cmd.CommandText = """
                INSERT INTO sensor (sensor_id, sensor_type, enabled, transport, p99_budget_ms, updated_at)
                VALUES ($id, $type, $enabled, $transport, $budget, $now)
                ON CONFLICT(sensor_id) DO UPDATE SET
                    sensor_type = excluded.sensor_type,
                    enabled = excluded.enabled,
                    transport = excluded.transport,
                    p99_budget_ms = excluded.p99_budget_ms,
                    updated_at = excluded.updated_at
            """;
            cmd.Parameters.AddWithValue("$id", s.SensorId);
            cmd.Parameters.AddWithValue("$type", s.SensorType);
            cmd.Parameters.AddWithValue("$enabled", s.Enabled ? 1 : 0);
            cmd.Parameters.AddWithValue("$transport", System.Text.Json.JsonSerializer.Serialize(s.Transport));
            cmd.Parameters.AddWithValue("$budget", s.P99BudgetMs);
            cmd.Parameters.AddWithValue("$now", DateTimeOffset.UtcNow.ToString("O"));
            cmd.ExecuteNonQuery();
        }
        // Similar upserts for protocol_config, calibration, alarm_rule.
        tx.Commit();
    }
}
```

`ON CONFLICT ... DO UPDATE` is idempotent and makes `--reseed` safe to rerun.

## Hot reload

| Table | Hot-reloadable | Mechanism |
|---|---|---|
| `sensor` (transport, enabled) | No | Restart the pipeline instance |
| `sensor_protocol_config` | No | Restart the pipeline instance |
| `sensor_calibration` | Yes | Push new values to the context cache |
| `sensor_alarm_rule` | Yes | Push new values to the context cache |

Admin endpoint on the Orchestration Manager triggers hot reload:

```csharp
app.MapPost("/api/v1/sensors/{id}/reload-calibration",
    async (string id, PipelineManager mgr, ISensorConfigRepository repo, CancellationToken ct) =>
    {
        var updated = await repo.LoadCalibrationAsync(id, ct);
        if (updated is null) return Results.NotFound();
        mgr.HotReloadCalibration(id, updated);
        return Results.NoContent();
    })
    .RequireAuthorization("admin");
```

## Migrations

If the orchestration manager uses EF Core already, reuse its migration flow for this schema — see `csharp-efcore-sqlite`.

If keeping raw `Microsoft.Data.Sqlite`, a simple version-based migrator is fine:

```csharp
public static class SchemaMigrator
{
    public static void Migrate(SqliteConnection conn)
    {
        using var versionCmd = conn.CreateCommand();
        versionCmd.CommandText = "PRAGMA user_version";
        var current = Convert.ToInt32(versionCmd.ExecuteScalar());

        if (current < 1) ApplyV1(conn);
        if (current < 2) ApplyV2(conn);
        // ...
    }

    private static void ApplyV1(SqliteConnection conn)
    {
        using var tx = conn.BeginTransaction();
        Exec(conn, """
            CREATE TABLE sensor (...);
            CREATE TABLE sensor_protocol_config (...);
            -- ...
        """);
        Exec(conn, "PRAGMA user_version = 1");
        tx.Commit();
    }

    private static void Exec(SqliteConnection conn, string sql)
    {
        using var cmd = conn.CreateCommand();
        cmd.CommandText = sql;
        cmd.ExecuteNonQuery();
    }
}
```

`PRAGMA user_version` is a SQLite-native integer stored per database. Simpler than a `__migrations` table for projects with a small, linear migration history.

## WAL mode

Enable write-ahead logging at first open:

```csharp
using var wal = conn.CreateCommand();
wal.CommandText = "PRAGMA journal_mode = WAL";
wal.ExecuteNonQuery();
```

WAL mode persists in the DB file. Set once. Gives readers and writers better concurrency — a reader doesn't block the writer, and vice versa. Important for the admin-endpoint writes that happen while the pipeline is running.
