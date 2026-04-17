# Testing

## Use SQLite in-memory, not the EF Core InMemory provider

The `Microsoft.EntityFrameworkCore.InMemory` provider is not a relational database. It doesn't enforce unique constraints, doesn't cascade, doesn't translate LINQ the same way as real SQL, and accepts queries that SQLite would reject.

**Tests that pass on InMemory and fail on SQLite are common. The reverse is rare.**

Use SQLite with `Data Source=:memory:`. It's the real provider, real SQL translation, fast enough.

## Fixture pattern

Base class, one fresh database per test:

```csharp
public abstract class DatabaseTest : IAsyncLifetime
{
    protected AppDbContext Db { get; private set; } = null!;
    private SqliteConnection _connection = null!;

    public async Task InitializeAsync()
    {
        _connection = new SqliteConnection("Data Source=:memory:");
        await _connection.OpenAsync();

        var options = new DbContextOptionsBuilder<AppDbContext>()
            .UseSqlite(_connection)
            .Options;

        Db = new AppDbContext(options);
        await Db.Database.EnsureCreatedAsync();
    }

    public async Task DisposeAsync()
    {
        await Db.DisposeAsync();
        await _connection.DisposeAsync();
    }
}
```

The connection is opened and held explicitly because SQLite's `:memory:` DB disappears as soon as the connection closes. Passing the open connection to `UseSqlite` keeps the DB alive for the test's lifetime.

`EnsureCreatedAsync` builds the schema from the current model — no migrations. Fast. For tests that must verify migrations work, use `MigrateAsync` instead; budget the extra time.

## A representative test

```csharp
public class OrderRepositoryTests : DatabaseTest
{
    [Fact]
    public async Task GetByCustomer_ReturnsOnlyThatCustomersOrders()
    {
        Db.Customers.Add(new Customer { Id = Guid.NewGuid(), Name = "Alice" });
        var alice = Db.Customers.Local.Single();
        Db.Customers.Add(new Customer { Id = Guid.NewGuid(), Name = "Bob" });
        var bob = Db.Customers.Local.Skip(1).Single();

        Db.Orders.AddRange(
            new Order { Id = Guid.NewGuid(), CustomerId = alice.Id, Total = 10m, CreatedAt = DateTime.UtcNow },
            new Order { Id = Guid.NewGuid(), CustomerId = alice.Id, Total = 20m, CreatedAt = DateTime.UtcNow },
            new Order { Id = Guid.NewGuid(), CustomerId = bob.Id,   Total = 30m, CreatedAt = DateTime.UtcNow });
        await Db.SaveChangesAsync();

        var repo = new OrderRepository(Db);
        var result = await repo.GetByCustomerAsync(alice.Id, CancellationToken.None);

        result.Should().HaveCount(2);
        result.Should().OnlyContain(o => o.CustomerId == alice.Id);
    }
}
```

## Test what matters

- **Filtering, ordering, pagination.** These have actual SQL translation behavior.
- **Cascade deletes.** Configured delete behavior produces real SQL; assert the outcome.
- **Unique constraints.** Attempting a duplicate should throw `DbUpdateException`.
- **Concurrency tokens.** Simulate a concurrent update, assert `DbUpdateConcurrencyException`.
- **Transactions.** A multi-step operation that fails mid-way must roll back fully.

## Don't test

- EF Core's change tracker.
- Serialization of JSON columns to JSON — tested upstream.
- Exact SQL text generated. `ToQueryString` is useful for debugging, brittle as an assertion.
- Simple property mappings. If `Order.Total` is a `decimal` mapped to a `TEXT` column, that works — don't test the round-trip for every property.

## Seeding helpers

For tests that all need a baseline fixture:

```csharp
public abstract class DatabaseTest : IAsyncLifetime
{
    protected AppDbContext Db { get; private set; } = null!;
    protected Customer Alice { get; private set; } = null!;
    protected Customer Bob { get; private set; } = null!;

    protected virtual async Task SeedAsync()
    {
        Alice = new Customer { Id = Guid.NewGuid(), Name = "Alice" };
        Bob   = new Customer { Id = Guid.NewGuid(), Name = "Bob" };
        Db.Customers.AddRange(Alice, Bob);
        await Db.SaveChangesAsync();
    }

    public async Task InitializeAsync()
    {
        // ... open connection, create schema as before
        await SeedAsync();
    }
}
```

Subclasses override `SeedAsync` to add test-specific data. Keeps each test's `Arrange` small.

## Transaction-per-test (alternative)

For a shared fixture — one DB across many tests — wrap each test in a transaction that rolls back:

```csharp
public class TransactionalTest : IAsyncLifetime
{
    private readonly DatabaseFixture _fixture;
    private IDbContextTransaction _tx = null!;
    protected AppDbContext Db { get; private set; } = null!;

    public TransactionalTest(DatabaseFixture fixture) => _fixture = fixture;

    public async Task InitializeAsync()
    {
        Db = _fixture.CreateContext();
        _tx = await Db.Database.BeginTransactionAsync();
    }

    public async Task DisposeAsync()
    {
        await _tx.RollbackAsync();
        await Db.DisposeAsync();
    }
}
```

Faster than rebuilding the schema per test, but subtle — any code under test that starts its own transaction breaks the pattern (SQLite doesn't support nested transactions; you get `TransactionScope`-style savepoints or outright errors).

For most projects, schema-per-test with in-memory SQLite is fast enough. Only move to transaction-per-test if the test suite is genuinely slow.

## Integration with the web host

When testing repositories through the full web stack, see the REST skill's testing reference for `WebApplicationFactory` patterns. Key SQLite-specific addition:

```csharp
builder.ConfigureServices(services =>
{
    services.RemoveAll<DbContextOptions<AppDbContext>>();

    var connection = new SqliteConnection("Data Source=:memory:");
    connection.Open();

    services.AddSingleton(connection);
    services.AddDbContext<AppDbContext>(options => options.UseSqlite(connection));
});
```

Registering the connection as a singleton keeps one shared in-memory DB alive for the whole `WebApplicationFactory` lifetime. Dispose the connection when the factory disposes.

## When SQLite isn't enough

Some features only exist (or only behave correctly) on a full RDBMS:

- Row-level locking semantics (SQLite is file-level).
- `SELECT ... FOR UPDATE` (SQLite ignores it).
- Window functions with complex partitioning (SQLite supports basic window functions, but edge cases differ).
- Full-text search with multiple languages (SQLite's FTS5 is solid but English-biased).
- True JSON indexing (SQLite's JSON is stored as TEXT; PostgreSQL `jsonb` is indexed natively).

If your production DB is Postgres or SQL Server, run integration tests against the real engine via Testcontainers:

```xml
<PackageReference Include="Testcontainers.PostgreSql" Version="4.*" />
```

```csharp
public sealed class PostgresFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public Task InitializeAsync() => _container.StartAsync();
    public Task DisposeAsync() => _container.DisposeAsync().AsTask();
}
```

Slower than SQLite (container startup is ~2-5 seconds), but catches provider-specific bugs that SQLite would hide. Share via `ICollectionFixture` so one container serves many tests.

This skill's default is SQLite everywhere — dev, test, and prod — so Testcontainers is rarely needed. Mention it because teams that later migrate to Postgres often need it.

## Speed

A healthy EF Core test suite with in-memory SQLite:

- Unit-level repository tests: 5-20ms each.
- Integration tests with `WebApplicationFactory`: 50-200ms each.
- Full project suite under 10 seconds for a few hundred tests.

If you're measurably slower, the usual culprits are `MigrateAsync` per test instead of `EnsureCreatedAsync`, a new `WebApplicationFactory` per test instead of `IClassFixture`, or tests that sleep. None of those are necessary.
