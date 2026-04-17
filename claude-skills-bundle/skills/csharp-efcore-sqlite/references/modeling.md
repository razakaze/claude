# Modeling

## Entities

Plain POCOs. No base class unless the team has chosen one (e.g., `Entity<TId>`). EF Core needs a parameterless constructor or a constructor it can bind — keep them simple.

```csharp
public sealed class Order
{
    public Guid Id { get; set; }
    public required Guid CustomerId { get; set; }
    public required DateTime CreatedAt { get; set; }
    public decimal Total { get; set; }
    public OrderStatus Status { get; set; } = OrderStatus.Pending;

    public Customer? Customer { get; set; }
    public List<OrderLine> Lines { get; set; } = [];
}
```

### Rules

- **`required` on non-navigation properties that must be set.** Compile-time enforcement.
- **Navigation properties nullable** (`Customer? Customer`) — EF Core populates them only when you `Include` them.
- **Initialize collection navigations to empty** — `List<OrderLine> Lines { get; set; } = [];`. Avoids null checks on every read.
- **Don't put logic on entities** unless the team uses a DDD-style rich domain model. For data-first patterns (which this skill defaults to), entities are data.

## Value objects (owned types)

For small composite values that live with the entity (money, address):

```csharp
public sealed record Money(decimal Amount, string Currency);

modelBuilder.Entity<Order>()
    .OwnsOne(o => o.Total, total =>
    {
        total.Property(m => m.Amount).HasColumnName("total_amount");
        total.Property(m => m.Currency).HasColumnName("total_currency").HasMaxLength(3);
    });
```

Owned types are stored as columns on the parent table, not a separate row. Clean in the model, no join on query.

## JSON columns

EF Core 10 supports JSON columns on SQLite. Useful for flexible payloads that don't need indexing.

```csharp
public sealed class Webhook
{
    public Guid Id { get; set; }
    public required string Url { get; set; }
    public WebhookConfig Config { get; set; } = new();
}

public sealed class WebhookConfig
{
    public int RetryCount { get; set; }
    public TimeSpan Timeout { get; set; }
    public List<string> Headers { get; set; } = [];
}

modelBuilder.Entity<Webhook>().OwnsOne(w => w.Config, cfg => cfg.ToJson());
```

Stored as `TEXT` JSON in a single column. Queryable via `w => w.Config.RetryCount > 3` — EF Core translates to SQLite's `json_extract`.

### When to use

- Config payloads, metadata, tags, flexible-schema fields.
- Fields you rarely filter on (filtering works but isn't indexed).

### When not to use

- Data you need to index — SQLite can't index JSON paths natively.
- Data you join on — JSON columns aren't relational.
- Data that has a clear schema and stable shape — that's what owned types are for.

## Enums

Store as string:

```csharp
public enum OrderStatus { Pending, Confirmed, Shipped, Cancelled }

modelBuilder.Entity<Order>()
    .Property(o => o.Status)
    .HasConversion<string>()
    .HasMaxLength(20);
```

Adding a new enum value doesn't break existing rows. Renaming a value is a data migration — plan for it.

Avoid storing enums as `int` — when someone deletes an enum value, the integer representations shift silently, corrupting the data.

## Inheritance

Three strategies in EF Core:

1. **TPH** (Table-per-Hierarchy) — single table with a discriminator. EF Core default. Fastest. Sparse columns for subtype-specific fields.
2. **TPT** (Table-per-Type) — base table plus one per subtype, joined on queries. Normalized. Slower.
3. **TPC** (Table-per-Concrete) — one table per concrete subtype, no base. Fastest for single-subtype queries, polymorphic queries union across tables.

**Default TPH unless you have a reason.** SQLite handles it well, and subtypes rarely benefit from separate tables.

```csharp
modelBuilder.Entity<Payment>()
    .HasDiscriminator<string>("payment_type")
    .HasValue<CardPayment>("card")
    .HasValue<BankTransferPayment>("bank")
    .HasValue<CashPayment>("cash");
```

## Shadow properties

Columns that exist in the DB but not on the entity class:

```csharp
modelBuilder.Entity<Order>()
    .Property<DateTime>("UpdatedAt")
    .HasDefaultValueSql("CURRENT_TIMESTAMP");
```

Access in queries: `EF.Property<DateTime>(order, "UpdatedAt")`.

Use sparingly. Shadow properties invisible to code hide intent. Use for audit columns, soft-delete flags — things that are infrastructure, not domain.

## Indexes

Declare every index explicitly. EF Core doesn't guess.

```csharp
builder.HasIndex(o => o.CustomerId);
builder.HasIndex(o => new { o.Status, o.CreatedAt });
builder.HasIndex(o => o.ExternalReference).IsUnique();
```

### Rules

- **Index every foreign key.** SQLite doesn't auto-index them.
- **Composite index column order matters.** Put the most-selective column first, or the column you filter on most.
- **Don't over-index.** Every index slows writes. Measure before adding.
- **Partial indexes** (`.HasFilter("status = 'Active'")`) for sparse columns with a common filter.

## Concurrency tokens

For optimistic concurrency, add a rowversion-like column:

```csharp
public sealed class Order
{
    // ...
    public uint Version { get; set; }
}

builder.Property(o => o.Version).IsRowVersion();
```

SQLite doesn't have native rowversion; EF Core increments it on save. `SaveChangesAsync` throws `DbUpdateConcurrencyException` if the row changed between load and save. Catch and retry or surface to the caller.

## Keys

- **`Guid` keys by default** for distributed-friendly IDs. Generated by the app (`Guid.NewGuid()`), `ValueGeneratedNever()` on the property config.
- **`int` auto-increment** for locally-generated IDs where guessability doesn't matter.
- **Composite keys** rarely — signals the entity is missing a natural single-column identity.

## Don't do

- **Lazy loading.** Either `Include` eagerly or project with `Select`. Lazy loading triggers N+1 queries silently.
- **Public setters on aggregate invariants.** If it matters that `Order.Status` transitions legally, expose a method, not a setter. (For anemic models this matters less — pick a style and hold.)
- **Two-way navigation you don't need.** Navigation in both directions doubles the mental model and rarely adds value. Keep the direction the domain actually uses.
