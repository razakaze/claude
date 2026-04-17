# Configuration: YAML → SQLite → Runtime

## The rule

YAML is seed. SQLite is runtime truth. **SQLite wins on conflict.** The only way YAML overwrites SQLite is an explicit `--reseed` flag.

## Startup flow

```
1. Open SQLite, run migrations.
2. For each sensor in YAML:
     - If sensor_id exists in SQLite → do nothing.
     - If not → insert from YAML.
3. Load all enabled sensors from SQLite.
4. Build one pipeline per row.
```

## Schema

Minimum tables. Extend per project; don't collapse.

```sql
CREATE TABLE sensor (
  sensor_id    TEXT PRIMARY KEY,
  sensor_type  TEXT NOT NULL,
  enabled      INTEGER NOT NULL DEFAULT 1,
  transport    TEXT NOT NULL,              -- JSON: {type: "udp", host: "...", port: ...}
  p99_budget_ms INTEGER NOT NULL,
  updated_at   TEXT NOT NULL
);

CREATE TABLE sensor_protocol_config (
  sensor_id TEXT PRIMARY KEY REFERENCES sensor(sensor_id) ON DELETE CASCADE,
  protocol  TEXT NOT NULL,
  params    TEXT NOT NULL                  -- JSON
);

CREATE TABLE sensor_calibration (
  sensor_id TEXT PRIMARY KEY REFERENCES sensor(sensor_id) ON DELETE CASCADE,
  params    TEXT NOT NULL,                 -- JSON
  updated_at TEXT NOT NULL
);

CREATE TABLE sensor_alarm_rule (
  rule_id   INTEGER PRIMARY KEY AUTOINCREMENT,
  sensor_id TEXT NOT NULL REFERENCES sensor(sensor_id) ON DELETE CASCADE,
  rule_type TEXT NOT NULL,
  params    TEXT NOT NULL,                 -- JSON
  enabled   INTEGER NOT NULL DEFAULT 1
);
```

JSON columns for `transport`, `params`, etc. — SQLite has no strong typing advantage, and the payloads are structurally variable per sensor type. Use Jackson to marshal these to typed records at read time.

## Migrations

Use Flyway with `db/migration/sqlite/` resources. Migrations are mandatory — no hand-edits to schema in production. SQLite Flyway support is first-class since Flyway 8.

## Hot reload

| Table | Hot-reloadable | Mechanism |
|---|---|---|
| `sensor` (transport, enabled) | No | Restart the pipeline instance |
| `sensor_protocol_config` | No | Restart the pipeline instance |
| `sensor_calibration` | Yes | Push to context cache |
| `sensor_alarm_rule` | Yes | Push to context cache |

Hot reload is triggered by an admin endpoint on the Orchestration Manager or by a file watcher on the SQLite file's `updated_at` columns (polling every N seconds from a dedicated scheduler — never from a pipeline stage).

## Access pattern

- **Startup reads:** plain JDBC, blocking, on the main thread. SQLite is fast enough that R2DBC adds no value here.
- **Runtime reads from pipeline stages:** none. Stages read from the context cache only. Calibration/rules loaded into the cache at pipeline start and on hot reload.
- **Writes (admin endpoint):** JDBC on `Schedulers.boundedElastic()`, bracketed in `Mono.fromCallable(...)`. This is fine — writes are rare and off the hot path.

## Seeder (sketch)

```java
public final class YamlSeeder {
    public int seed(Path yamlFile, DataSource ds, boolean reseed) {
        List<SensorConfig> fromYaml = parseYaml(yamlFile);
        try (Connection c = ds.getConnection()) {
            int inserted = 0;
            for (SensorConfig s : fromYaml) {
                if (reseed || !exists(c, s.sensorId())) {
                    upsert(c, s);
                    inserted++;
                }
            }
            return inserted;
        }
    }
}
```

Idempotent. Runs once at startup with `reseed=false`. The CLI flag flips to `true`.
