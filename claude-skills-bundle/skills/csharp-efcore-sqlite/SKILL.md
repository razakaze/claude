---
name: csharp-efcore-sqlite
description: Use when writing or modifying data access code in C# 14 / .NET 10 that uses Entity Framework Core 10 against SQLite. Triggers on mentions of DbContext, DbSet, OnModelCreating, migrations (Add-Migration, dotnet ef), AsNoTracking, IQueryable, IDbContextFactory, SaveChanges, transactions, JournalMode WAL, PRAGMA, or any .csproj referencing Microsoft.EntityFrameworkCore.Sqlite. Use whenever the user is adding an entity, writing a query, creating a migration, designing a unit of work, handling SQLite-specific constraints (schema alteration, concurrency, type affinity), or testing with in-memory SQLite. Consult even when the user doesn't name these terms — any repository class with an injected DbContext or query composition against a DbSet is in scope.
---

# C# EF Core with SQLite (EF Core 10 / .NET 10)

Use **code-first migrations**. **`IDbContextFactory<T>`** for short-lived contexts in long-running hosts (services, background workers). **`AsNoTracking`** as the default for reads. **Explicit transactions** at the unit-of-work boundary. Respect **SQLite's schema-alteration limits** — plan migrations accordingly.

SQLite is not SQL Server. Don't use EF Core patterns that assume it is.

---

## Project shape

### csproj

```xml
<ItemGroup>
  <PackageReference Include="Microsoft.EntityFrameworkCore" Version="10.0.*" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.*" />
  <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.*">
    <PrivateAssets>all</PrivateAssets>
    <IncludeAssets>runtime; build; native; contentfiles; analyzers; buildtransitive</IncludeAssets>
  </PackageReference>
</ItemGroup>
```

`Microsoft.EntityFrameworkCore.Design` is needed for the `dotnet ef` tools. Keep it `PrivateAssets=all` so it doesn't ship transitively.

### DbContext

```csharp
public sealed class AppDbContext(DbContextOptions<AppDbContext> options) : DbContext(options)
{
    public DbSet<Order> Orders => Set<Order>();
    public DbSet<Customer> Customers => Set<Customer>();
    public DbSet<OrderLine> OrderLines => Set<OrderLine>();

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.ApplyConfigurationsFromAssembly(typeof(AppDbContext).Assembly);
    }
}
```

### Configuration per entity

```csharp
public sealed class OrderConfiguration : IEntityTypeConfiguration<Order>
{
    public void Configure(EntityTypeBuilder<Order> builder)
    {
        builder.ToTable("orders");

        builder.HasKey(o => o.Id);
        builder.Property(o => o.Id).ValueGeneratedNever();  // Guid, set by app

        builder.Property(o => o.Total).HasColumnType("TEXT");  // SQLite decimal storage
        builder.Property(o => o.Status).HasConversion<string>();  // enum as string

        builder.HasOne(o => o.Customer)
               .WithMany(c => c.Orders)
               .HasForeignKey(o => o.CustomerId)
               .OnDelete(DeleteBehavior.Restrict);

        builder.HasMany(o => o.Lines)
               .WithOne()
               .HasForeignKey(l => l.OrderId)
               .OnDelete(DeleteBehavior.Cascade);

        builder.HasIndex(o => o.CustomerId);
        builder.HasIndex(o => new { o.Status, o.CreatedAt });
    }
}
```

### Rules

- **One `IEntityTypeConfiguration<T>` per aggregate root.** `ApplyConfigurationsFromAssembly` finds them all; don't pile configuration into `OnModelCreating`.
- **Explicit table names in `snake_case` lowercase** — or whatever the team convention is. Default PascalCase tables clash with SQL style in the DB.
- **`DeleteBehavior` is explicit.** SQLite respects it when foreign keys are enabled (they are, by default in EF Core's SQLite provider).
- **Indexes on foreign keys and on every WHERE clause column** that isn't the primary key. SQLite doesn't index FKs automatically.
- **Enums stored as strings** (`HasConversion<string>`) for forward compatibility. Adding a new enum value doesn't require re-numbering.

See `references/modeling.md` for value objects, owned types, JSON columns, shadow properties, and inheritance strategies.

---

## Registration

### In a web app (scoped DbContext)

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
{
    options.UseSqlite(builder.Configuration.GetConnectionString("Default"),
        sqlite => sqlite.CommandTimeout(30));

    if (builder.Environment.IsDevelopment())
    {
        options.EnableDetailedErrors();
        options.EnableSensitiveDataLogging();
    }
});
```

Scoped lifetime, one `DbContext` per request. Standard for ASP.NET Core.

### In a Windows Service or Avalonia app (factory)

```csharp
builder.Services.AddDbContextFactory<AppDbContext>(options =>
{
    options.UseSqlite(builder.Configuration.GetConnectionString("Default"));
});
```

Long-running hosts have no per-request scope. `AddDbContextFactory` lets you create short-lived contexts explicitly at the unit-of-work boundary:

```csharp
public sealed class OrderService(IDbContextFactory<AppDbContext> factory)
{
    public async Task<OrderDto> GetAsync(Guid id, CancellationToken ct)
    {
        await using var db = await factory.CreateDbContextAsync(ct);
        return await db.Orders
            .AsNoTracking()
            .Where(o => o.Id == id)
            .Select(o => new OrderDto(o.Id, o.Total, o.Status))
            .FirstOrDefaultAsync(ct)
            ?? throw new KeyNotFoundException();
    }
}
```

### Connection string

```json
{
  "ConnectionStrings": {
    "Default": "Data Source=app.db;Cache=Shared;Foreign Keys=True;Pooling=True"
  }
}
```

- **`Cache=Shared`** — multiple connections share one in-memory page cache. Helpful for in-process concurrency.
- **`Foreign Keys=True`** — EF Core sets this by default, but stating it explicitly documents intent.
- **`Pooling=True`** — connection pooling. On by default; states intent.

---

## Queries

### Reads: AsNoTracking

```csharp
var orders = await db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId)
    .Select(o => new OrderDto(o.Id, o.Total, o.Status))
    .ToListAsync(ct);
```

### Rules

- **`AsNoTracking` for reads that don't update.** Tracking builds the change tracker for every entity — pure overhead if you're not saving.
- **Project with `Select`** into DTOs at the query boundary. Don't load full entities and map outside — the query pulls less data.
- **`ToListAsync` / `FirstOrDefaultAsync` / `SingleOrDefaultAsync` — always async.** The sync versions block threads; in a web or service host, that's a scaling bug.
- **Pagination via `Skip`/`Take` with a stable order.** `OrderBy` is mandatory — SQLite doesn't guarantee insertion order.
- **`AsSplitQuery()` for collections with large joins.** Single-query Cartesian explosion with multiple `Include` is the #1 EF Core perf mistake.

### Writes

```csharp
public async Task<Guid> CreateOrderAsync(CreateOrderRequest request, CancellationToken ct)
{
    await using var db = await factory.CreateDbContextAsync(ct);

    var order = new Order
    {
        Id = Guid.NewGuid(),
        CustomerId = request.CustomerId,
        CreatedAt = DateTime.UtcNow,
        Lines = request.Lines.Select(l => new OrderLine
        {
            Id = Guid.NewGuid(),
            ProductId = l.ProductId,
            Quantity = l.Quantity,
        }).ToList(),
    };

    db.Orders.Add(order);
    await db.SaveChangesAsync(ct);
    return order.Id;
}
```

### Rules

- **One `SaveChanges` per unit of work.** Don't call it in a loop — batch.
- **Explicit transactions for multi-step work** that must be atomic beyond one `SaveChanges`. `db.Database.BeginTransactionAsync(ct)`.
- **Don't load entities just to update a few fields.** `ExecuteUpdateAsync` and `ExecuteDeleteAsync` (EF Core 7+) skip the change tracker:
  ```csharp
  await db.Orders
      .Where(o => o.Status == OrderStatus.Pending && o.CreatedAt < cutoff)
      .ExecuteUpdateAsync(s => s.SetProperty(o => o.Status, OrderStatus.Expired), ct);
  ```

See `references/queries.md` for LINQ patterns, raw SQL, compiled queries, and query performance debugging.

---

## Migrations

### Creating

```cmd
dotnet ef migrations add InitialCreate
dotnet ef migrations add AddOrderStatus
```

Migration names are verbs: what this migration does. `InitialCreate`, `AddOrderStatus`, `RenameCustomerEmail`. Not `Migration1`, `Fix`, `Update2`.

### Applying

At startup, apply pending migrations once:

```csharp
using (var scope = app.Services.CreateScope())
{
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    await db.Database.MigrateAsync();
}
```

For services that deploy many instances, let only one apply migrations (leader election, or a separate migration job). Concurrent `MigrateAsync` calls can deadlock.

### SQLite schema-alteration limits

SQLite supports a narrow set of `ALTER TABLE` operations:

| Operation | SQLite support |
|---|---|
| `ADD COLUMN` | Yes |
| `DROP COLUMN` | SQLite 3.35+ (EF Core handles) |
| `ALTER COLUMN` type/nullability | **No** — EF Core workarounds with table rebuild |
| `RENAME COLUMN` | SQLite 3.25+ |
| `RENAME TABLE` | Yes |

For unsupported ops, EF Core emits a **table rebuild** migration: create new table, copy data, drop old, rename new. This works but is slow on large tables and destroys implicit row order.

**Plan for this.** Get the column types right early. Widening `int` to `long` or narrowing `VARCHAR(100)` to `VARCHAR(50)` triggers a rebuild. Prefer column addition over column modification.

### What a migration does not do

- **Seed data** for production. Seeding via `HasData` is model-static; for real seed data (reference tables, defaults), write a separate seeder that runs at startup and uses idempotent inserts.
- **Index tuning** well. EF Core creates the indexes you declared. Production query tuning is a separate, ongoing concern.

See `references/migrations.md` for complex migrations (data transformations, renames), rollback strategy, and the design-time factory pattern.

---

## Transactions

EF Core wraps each `SaveChangesAsync` in a transaction implicitly. For multi-step or cross-context work, start explicit:

```csharp
await using var tx = await db.Database.BeginTransactionAsync(ct);
try
{
    db.Orders.Add(order);
    await db.SaveChangesAsync(ct);

    var stock = await stockContext.Stock.FirstAsync(s => s.ProductId == productId, ct);
    stock.Quantity -= order.Lines.Sum(l => l.Quantity);
    await stockContext.SaveChangesAsync(ct);

    await tx.CommitAsync(ct);
}
catch
{
    await tx.RollbackAsync(ct);
    throw;
}
```

### SQLite concurrency

SQLite uses **file locking**. Only one writer at a time; readers don't block, but a writer blocks other writers. Under contention you get `SQLITE_BUSY` errors.

**WAL mode** (Write-Ahead Logging) helps: writers don't block readers and vice versa. Enable once:

```csharp
public static async Task EnableWalAsync(this DbContext context, CancellationToken ct)
{
    await context.Database.ExecuteSqlRawAsync("PRAGMA journal_mode = WAL;", ct);
}
```

Call from the app's startup, once. WAL mode persists in the DB file.

**Busy timeout** — tell SQLite to wait on lock contention instead of failing immediately:

```csharp
options.UseSqlite(connStr, sqlite => sqlite.CommandTimeout(30))
       .ConfigureWarnings(w => w.Throw(RelationalEventId.MultipleCollectionIncludeWarning));
```

Or via connection string: `Data Source=app.db;Default Timeout=30`.

---

## SQLite quirks

### Type affinity

SQLite is dynamically typed. `INTEGER`, `REAL`, `TEXT`, `BLOB`, `NUMERIC`. EF Core maps .NET types:

| .NET | SQLite |
|---|---|
| `int`, `long`, `bool`, `DateTime`, `DateOnly`, `TimeOnly`, `Guid` | `INTEGER` (or `TEXT` for Guid) |
| `float`, `double` | `REAL` |
| `decimal` | `TEXT` (stored as string — exact decimal preserved) |
| `string`, enum-as-string | `TEXT` |
| `byte[]` | `BLOB` |

**Decimals as TEXT** — this is correct. `REAL` would lose precision. Arithmetic in SQL on decimals works via CAST, but prefer filtering/ordering in C# for decimals if you hit precision issues.

**DateTime as TEXT** for ISO 8601 by default. Use `DateTimeOffset` if you need timezone awareness — otherwise you'll regret it the first time DST changes.

### Limited collation

SQLite has three built-in collations: `BINARY`, `NOCASE`, `RTRIM`. No Unicode-aware case-insensitive matching by default. For user-visible searches, do matching in C# after pulling data, or set up ICU extension (rare).

### FTS for search

Full-text search via `FTS5` is SQLite's strength. Set up a contentless FTS5 table mirroring the rows you search, populate via triggers. Not free — but orders of magnitude faster than `LIKE '%...%'`. EF Core doesn't model FTS natively; use `FromSqlRaw` for FTS queries.

### File-based or in-memory

- **File:** `Data Source=app.db` — standard, persistent.
- **Shared in-memory:** `Data Source=file::memory:?cache=shared` — the DB lives until the last connection closes. Multiple connections see the same DB.
- **Private in-memory:** `Data Source=:memory:` — one connection only. If the connection closes, data is gone.

For tests: shared in-memory with `OpenConnection` to pin the first connection open.

---

## Testing

### In-memory SQLite

Most realistic option for tests — real SQL translation, real transactions. Fast.

```csharp
public abstract class DatabaseTestBase : IAsyncLifetime
{
    protected AppDbContext Db { get; private set; } = null!;

    public async Task InitializeAsync()
    {
        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite("Data Source=:memory:")
            .Options;

        Db = new AppDbContext(options);
        await Db.Database.OpenConnectionAsync();
        await Db.Database.EnsureCreatedAsync();
    }

    public Task DisposeAsync()
    {
        Db.Dispose();
        return Task.CompletedTask;
    }
}
```

`OpenConnectionAsync` pins the connection so the DB survives until dispose. `EnsureCreatedAsync` builds the schema from the current model — fast, but doesn't test migrations.

### For migration tests

```csharp
await Db.Database.MigrateAsync();
```

Slower. Use selectively, not per test — usually in a single "schema roundtrips cleanly" test.

### EF Core In-Memory provider — avoid

`Microsoft.EntityFrameworkCore.InMemory` is not a real SQL engine. It doesn't enforce referential integrity, doesn't translate LINQ the same way, doesn't catch provider-specific query errors. Tests pass that real SQLite would fail.

**Use SQLite in-memory. Not InMemory provider.**

### What to test

- Repository methods: filtering, pagination, ordering.
- Transactions: rollback correctly on failure.
- Unique constraints and cascade deletes behave as configured.
- Migrations apply cleanly to an empty DB (at least once).

### What not to test

- EF Core's change tracker. It works.
- LINQ translation of simple queries. If you're writing a complex LINQ expression and asserting the SQL it produces, the test is brittle and the query might belong as raw SQL anyway.

See `references/testing.md` for transaction-per-test isolation, fixture sharing, and Testcontainers with Postgres when SQLite isn't enough.

---

## C# 14 idioms worth using here

- **Primary constructors** on `DbContext` (`: DbContext(options)`) and repositories.
- **Collection expressions** (`[]`) for default empty navigation collections.
- **`required` members** on entities for non-nullable reference properties — forces initialization, prevents silent nulls.
- **`field` keyword** for properties that do light normalization in setters (trimming, casing).

---

## What this skill does not cover

- **Non-SQLite providers.** SQL Server, PostgreSQL, MySQL have different perf profiles, type mappings, and feature support. Some patterns here (decimal-as-TEXT, WAL mode, FTS5) are SQLite-only.
- **EF Core on mobile** (Xamarin, MAUI). Similar API, different deployment story.
- **CQRS / event sourcing on EF Core.** Different architecture; not covered here.
- **Raw ADO.NET / Dapper.** When EF Core is wrong (hot-path queries where ORM overhead matters), drop to Dapper — but that's a different skill.
- **Bulk operations beyond `ExecuteUpdate`/`ExecuteDelete`.** For million-row imports, SQLite's `INSERT` with prepared statements outside EF Core is faster; use it.
