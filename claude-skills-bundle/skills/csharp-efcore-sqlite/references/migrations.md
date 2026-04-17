# Migrations

## Commands

```cmd
dotnet ef migrations add AddOrderStatus
dotnet ef migrations remove
dotnet ef migrations script --idempotent --output migrate.sql
dotnet ef database update
dotnet ef database update PreviousMigrationName   :: rollback
```

Run from the project that holds the `DbContext`, or use `--project` / `--startup-project` flags when the context and the host live in different projects.

## Naming

Migration names are verbs that describe the intent, not the mechanism.

| Good | Bad |
|---|---|
| `AddOrderStatus` | `Migration2` |
| `RenameCustomerEmailColumn` | `FixCustomer` |
| `AddIndexOrderCreatedAt` | `Update1` |
| `SeedCountries` | `InitData` |

A migration name is the first thing a reviewer reads six months later. Make it tell the truth.

## Design-time factory

For any non-trivial host (web app, Windows Service, Avalonia), the EF tools can't always construct the `DbContext` automatically at design time — they don't know how to build your DI container. Give them an explicit factory:

```csharp
public sealed class AppDbContextFactory : IDesignTimeDbContextFactory<AppDbContext>
{
    public AppDbContext CreateDbContext(string[] args)
    {
        var builder = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite("Data Source=design-time.db");
        return new AppDbContext(builder.Options);
    }
}
```

The connection string here is only used at design time — for reading the model and generating SQL. The database file doesn't need to exist. Never put production credentials here.

## One migration per concern

A migration should do one thing. Adding a column + renaming a table + seeding data is three migrations, not one. When it goes wrong in production and you need to revert just the rename, you can't if it's bundled.

```
20260417101500_AddOrderStatus
20260417102000_AddIndexOrderStatusCreatedAt
20260417103000_SeedOrderStatusDefaults
```

## Data migrations

EF Core migrations are schema-first. For data transformation, put the SQL inside the migration's `Up`:

```csharp
public partial class BackfillOrderStatusFromLegacyField : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql(@"
            UPDATE orders
            SET status = CASE legacy_status_code
                WHEN 0 THEN 'Pending'
                WHEN 1 THEN 'Confirmed'
                WHEN 2 THEN 'Shipped'
                WHEN 3 THEN 'Cancelled'
            END
            WHERE status IS NULL
        ");
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.Sql("UPDATE orders SET status = NULL");
    }
}
```

### Rules

- **Raw SQL, not entity manipulation**, in migrations. The entity model at migration time is whatever the code currently says — it may not match what the DB schema looked like when the migration was written. Raw SQL is stable.
- **Idempotent data migrations where possible.** Running twice should leave the DB in the same state as running once.
- **Separate schema and data migrations.** A schema migration that also moves data is harder to review and harder to roll back.

## SQLite-specific gotchas

### Table rebuilds

When EF Core generates a migration for an unsupported ALTER (changing a column type, altering nullability, dropping a constraint), it emits a **table rebuild**:

1. Create `__temp_orders` with the new schema.
2. Copy rows from `orders` into `__temp_orders`.
3. Drop `orders`.
4. Rename `__temp_orders` → `orders`.
5. Recreate indexes and foreign keys.

On a small table this is invisible. On a 10-million-row table it locks writes for minutes and doubles disk use during the copy.

**Before merging a migration, check whether it rebuilds tables.** Look at the generated migration code — if you see a `CreateTable` + `Sql("INSERT INTO ... SELECT FROM ...")` + `DropTable` sequence, that's a rebuild.

Mitigations:
- Get types right early. Adding columns is cheap, changing types is not.
- For big tables, script a custom rebuild that uses batched inserts and a progress log, and mark the EF migration as "applied manually" (`dotnet ef migrations script` to see the SQL, run it yourself, then `__EFMigrationsHistory` insert).

### Foreign keys during migration

SQLite enforces foreign keys per connection. EF Core disables them during table rebuilds (necessary) and re-enables after. If a migration fails partway, foreign keys may be left off on that connection — next startup re-enables them.

### Composite PK / unique constraint changes

Dropping or altering a unique constraint also triggers a table rebuild. Budget accordingly.

## Applying migrations

### At startup (single-instance hosts)

```csharp
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}
```

Fine for single-instance services (Windows Service, desktop, dev). Done once on boot, before traffic.

### Multi-instance or production rollouts

Don't call `MigrateAsync` from every replica. Race conditions and locking will bite.

Options:

1. **Separate migration job.** A standalone process (container, one-shot script, CI step) applies migrations. App replicas start after migration completes. Cleanest.
2. **Leader election.** Only one replica applies migrations. Requires a coordination mechanism (file lock on the DB file, distributed lock). For SQLite specifically, a file lock on the DB is natural.
3. **Generated SQL script.** `dotnet ef migrations script --idempotent` produces a SQL file. DBA or deploy script runs it. Audit trail, no runtime surprises.

For SQLite-based services the simplest pattern is option 2 with a file lock on the DB file — only one instance can hold the lock, it applies the migration, releases the lock, everyone else opens the connection after.

## Rollback

EF Core migrations have `Down` methods. In practice, rolling back a migration in production is rarely a good idea:

- Schema rollback can destroy data (e.g., dropping a column removes the data in it).
- Application code expects the newer schema — running old code against the old schema might work, but the transition is brittle.

**Prefer roll-forward.** If a migration causes a problem, write a new migration that fixes the problem, don't revert the old one. `Down` is for local development, not production ops.

### When rollback is legitimate

- Dev/test environments — back and forth freely.
- Immediately after deploy, before any write traffic, when you discover the migration itself was wrong.

## Squashing

After many migrations accumulate, some teams squash old migrations into a single "initial" migration. EF Core doesn't support this directly — you'd:

1. Archive the old migration files.
2. Drop the `__EFMigrationsHistory` table.
3. Generate a new initial migration from the current model.
4. Mark it as applied without running (for existing databases).

This is disruptive. Only worth it if migrations number in the hundreds and the folder is genuinely noisy. Most projects never need to.

## What not to do

- **Don't check migrations into the repo inconsistently with the model snapshot.** The `ModelSnapshot.cs` file next to migrations is part of the migration; if it's out of sync, the next migration's diff is wrong. Commit both or neither.
- **Don't edit a migration after it's been applied anywhere.** If a production server has run migration X, editing X retroactively means that server's DB schema doesn't match what anyone else's server would have. Add a new migration instead.
- **Don't rely on `EnsureCreated` in production.** It bypasses migrations entirely — once you've used it, you can't cleanly move to migrations. `EnsureCreated` is for tests only.
- **Don't mix manual schema changes with migrations.** If ops applies a hotfix schema change directly to production, the next EF migration may conflict. Hotfix by writing a migration and deploying it, even if it's faster to just `ALTER TABLE`.
