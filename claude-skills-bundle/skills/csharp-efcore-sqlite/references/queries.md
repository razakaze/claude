# Queries

## The canonical read

```csharp
var orders = await db.Orders
    .AsNoTracking()
    .Where(o => o.CustomerId == customerId && o.Status == OrderStatus.Confirmed)
    .OrderByDescending(o => o.CreatedAt)
    .Take(50)
    .Select(o => new OrderDto(o.Id, o.Total, o.Status, o.CreatedAt))
    .ToListAsync(ct);
```

- `AsNoTracking` — no change tracker.
- `Where` first, narrow early.
- `OrderBy` before `Take` — SQLite doesn't preserve insertion order.
- `Select` into a DTO — pulls only the columns the DTO needs.
- `ToListAsync` with cancellation token.

## Projection

Project into DTOs at the query boundary. The ORM should produce the shape your caller wants.

```csharp
// GOOD — one query, few columns
var summaries = await db.Orders
    .AsNoTracking()
    .Where(...)
    .Select(o => new OrderSummary(o.Id, o.Total, o.Customer!.Name))
    .ToListAsync(ct);

// BAD — loads full entities, N+1 on Customer navigation
var orders = await db.Orders.Where(...).ToListAsync(ct);
var summaries = orders.Select(o => new OrderSummary(o.Id, o.Total, o.Customer.Name));
```

## Include and split queries

When you genuinely need the full entity graph:

```csharp
var order = await db.Orders
    .AsNoTracking()
    .Include(o => o.Lines)
    .Include(o => o.Customer)
    .AsSplitQuery()
    .FirstOrDefaultAsync(o => o.Id == id, ct);
```

**`AsSplitQuery` when you `Include` a collection.** Without it, EF Core builds one SQL query with Cartesian joins — 100 orders × 10 lines × 3 tags = 3000 rows transferred for 100 logical objects. Split query runs three queries and composes in memory. Nearly always faster for non-trivial includes.

**Don't `Include` when you can `Select`.** A DTO projection with `o => new { o.Total, Lines = o.Lines.Select(...) }` is usually cleaner and generates better SQL.

## Filtering

### In SQL, not C#

```csharp
// GOOD — translated to SQL
await db.Orders.Where(o => o.Total > 100m).ToListAsync(ct);

// BAD — loads everything, filters in memory
(await db.Orders.ToListAsync(ct)).Where(o => o.Total > 100m);
```

If the expression can't be translated, EF Core throws (since EF Core 3.0 — no more silent client evaluation). Good. Read the error and rewrite the query.

### Null handling

```csharp
await db.Orders.Where(o => o.Notes == null).ToListAsync(ct);   // IS NULL
await db.Orders.Where(o => o.Notes != null).ToListAsync(ct);   // IS NOT NULL
await db.Orders.Where(o => o.Notes == "x" || o.Notes == null).ToListAsync(ct);
```

EF Core emits the right SQL for nullable comparisons. Don't write `== "" ?? true`-style dances.

### String matching

```csharp
// Prefix
await db.Orders.Where(o => o.ExternalId.StartsWith("A-")).ToListAsync(ct);

// Case-insensitive substring
await db.Orders.Where(o => EF.Functions.Like(o.Notes, "%urgent%")).ToListAsync(ct);
```

SQLite `LIKE` is case-insensitive for ASCII by default. For Unicode-correct matching, set up ICU extension — or do filtering in C# after loading, if the result set is small.

## Pagination

```csharp
public async Task<PagedResult<OrderDto>> GetPagedAsync(int page, int pageSize, CancellationToken ct)
{
    var query = db.Orders
        .AsNoTracking()
        .Where(o => o.Status != OrderStatus.Cancelled)
        .OrderByDescending(o => o.CreatedAt)
        .ThenBy(o => o.Id);  // stable tie-break

    var total = await query.CountAsync(ct);
    var items = await query
        .Skip((page - 1) * pageSize)
        .Take(pageSize)
        .Select(o => new OrderDto(o.Id, o.Total, o.Status, o.CreatedAt))
        .ToListAsync(ct);

    return new PagedResult<OrderDto>(items, total, page, pageSize);
}
```

`ThenBy(o => o.Id)` — tie-break with a unique column so ordering is deterministic across pages.

For large tables, cursor-based pagination (`WHERE CreatedAt < lastSeen`) beats offset-based — `Skip(100000)` makes SQLite scan 100000 rows.

## Aggregations

```csharp
var stats = await db.Orders
    .AsNoTracking()
    .Where(o => o.CreatedAt >= since)
    .GroupBy(o => o.CustomerId)
    .Select(g => new CustomerStats(
        g.Key,
        g.Count(),
        g.Sum(o => o.Total),
        g.Max(o => o.CreatedAt)))
    .ToListAsync(ct);
```

EF Core translates to a single `GROUP BY` SQL query. No client-side aggregation.

## Writes

### Single entity

```csharp
var order = new Order { Id = Guid.NewGuid(), /* ... */ };
db.Orders.Add(order);
await db.SaveChangesAsync(ct);
```

### Bulk update without loading

```csharp
await db.Orders
    .Where(o => o.Status == OrderStatus.Pending && o.CreatedAt < cutoff)
    .ExecuteUpdateAsync(s => s
        .SetProperty(o => o.Status, OrderStatus.Expired)
        .SetProperty(o => o.UpdatedAt, DateTime.UtcNow), ct);
```

Generates a single `UPDATE`, no change tracking, no entity materialization. Use when you're updating many rows by criteria.

### Bulk delete

```csharp
await db.Orders
    .Where(o => o.Status == OrderStatus.Cancelled && o.CreatedAt < cutoff)
    .ExecuteDeleteAsync(ct);
```

Skips soft-delete filters and any `SaveChanges` interceptors. Don't use if those matter to you.

### Batch inserts

```csharp
db.Orders.AddRange(newOrders);
await db.SaveChangesAsync(ct);
```

EF Core batches INSERTs. For tens of rows, fine. For tens of thousands, drop to raw SQL or Dapper — EF Core's per-entity tracking cost is real at scale.

## Raw SQL

When LINQ can't express it or the generated SQL is bad:

```csharp
var orders = await db.Orders
    .FromSqlInterpolated($@"
        SELECT * FROM orders
        WHERE customer_id = {customerId}
        AND EXISTS (SELECT 1 FROM order_lines l WHERE l.order_id = orders.id AND l.quantity > {minQty})")
    .AsNoTracking()
    .ToListAsync(ct);
```

`FromSqlInterpolated` parameterizes safely. **Never** string-concatenate user input into a SQL query.

For non-entity projections:

```csharp
var result = await db.Database
    .SqlQueryRaw<MyDto>("SELECT id AS Id, name AS Name FROM orders WHERE ...")
    .ToListAsync(ct);
```

## Compiled queries

For hot-path queries that run thousands of times per second:

```csharp
private static readonly Func<AppDbContext, Guid, CancellationToken, Task<Order?>> GetOrderById =
    EF.CompileAsyncQuery((AppDbContext db, Guid id, CancellationToken ct) =>
        db.Orders.AsNoTracking().FirstOrDefault(o => o.Id == id));

// usage
var order = await GetOrderById(db, id, ct);
```

Skips LINQ expression tree construction on every call. Measure before reaching for this — the win is 10-20% on a hot path, nothing on a cold one.

## Debugging slow queries

### Log the SQL

```csharp
options.UseSqlite(connStr).LogTo(Console.WriteLine, LogLevel.Information);
```

In development, this shows every query with parameters. Read them. Query that takes 500ms but LINQ looks fine? It's probably generating a Cartesian join — add `AsSplitQuery`.

### `ToQueryString()`

```csharp
var sql = db.Orders.Where(o => o.Total > 100m).ToQueryString();
```

Returns the SQL without executing. Useful in tests and for offline analysis.

### EXPLAIN QUERY PLAN

SQLite has a query planner — run:

```sql
EXPLAIN QUERY PLAN SELECT * FROM orders WHERE customer_id = ?
```

If the plan shows `SCAN`, you're missing an index. `SEARCH ... USING INDEX ...` means the index is being used.
