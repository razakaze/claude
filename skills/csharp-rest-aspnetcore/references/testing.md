# Testing

## Three layers

1. **Endpoint integration tests** via `WebApplicationFactory<Program>`. Real pipeline, test server, mocked dependencies.
2. **Service unit tests.** Plain xUnit/NUnit.
3. **Contract tests** (optional, for published APIs). Verify OpenAPI spec hasn't drifted.

## Make Program accessible

```csharp
// at bottom of Program.cs
public partial class Program { }
```

Without this, `WebApplicationFactory<Program>` can't find the entry class.

## Integration test setup

```csharp
public class OrdersEndpointsTests : IClassFixture<WebApplicationFactory<Program>>
{
    private readonly WebApplicationFactory<Program> _factory;

    public OrdersEndpointsTests(WebApplicationFactory<Program> factory)
    {
        _factory = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                var repo = Substitute.For<IOrderRepository>();
                services.Replace(ServiceDescriptor.Singleton(repo));
            });
        });
    }
}
```

`Replace` overrides the real registration. Each test gets a fresh scope via `_factory.CreateClient()`.

## Authenticated requests

Bearer token tests shouldn't call a real IdP. Stub the auth scheme:

```csharp
public sealed class TestAuthHandler(
    IOptionsMonitor<AuthenticationSchemeOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder)
    : AuthenticationHandler<AuthenticationSchemeOptions>(options, logger, encoder)
{
    public const string Scheme = "Test";

    protected override Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        var claims = new[]
        {
            new Claim("sub", "test-user"),
            new Claim("scope", "orders:read orders:write"),
            new Claim(ClaimTypes.Role, "admin"),
        };
        var identity = new ClaimsIdentity(claims, Scheme);
        var principal = new ClaimsPrincipal(identity);
        var ticket = new AuthenticationTicket(principal, Scheme);
        return Task.FromResult(AuthenticateResult.Success(ticket));
    }
}
```

Register in the test factory:

```csharp
builder.ConfigureServices(services =>
{
    services.Configure<AuthenticationOptions>(options =>
    {
        options.DefaultAuthenticateScheme = TestAuthHandler.Scheme;
        options.DefaultChallengeScheme = TestAuthHandler.Scheme;
    });
    services.AddAuthentication(TestAuthHandler.Scheme)
        .AddScheme<AuthenticationSchemeOptions, TestAuthHandler>(TestAuthHandler.Scheme, _ => { });
});
```

Now every test request is authenticated with the stub identity. Vary claims per test by making `TestAuthHandler` read from an injected context.

## Database fixtures

For tests that need EF Core, use SQLite in-memory or Testcontainers depending on realism needs.

### SQLite in-memory (fast, no Docker)

```csharp
builder.ConfigureServices(services =>
{
    services.RemoveAll<DbContextOptions<AppDbContext>>();
    services.AddDbContext<AppDbContext>(options =>
        options.UseSqlite("Data Source=:memory:"));

    var provider = services.BuildServiceProvider();
    using var scope = provider.CreateScope();
    var db = scope.ServiceProvider.GetRequiredService<AppDbContext>();
    db.Database.OpenConnection();
    db.Database.EnsureCreated();
});
```

Works for most tests. Fails for SQL Server-specific features (no CLR functions, limited geography support).

### Testcontainers (realistic, slower)

```xml
<PackageReference Include="Testcontainers.PostgreSql" Version="4.*" />
```

```csharp
public sealed class DatabaseFixture : IAsyncLifetime
{
    private readonly PostgreSqlContainer _container = new PostgreSqlBuilder()
        .WithImage("postgres:16-alpine")
        .Build();

    public string ConnectionString => _container.GetConnectionString();

    public Task InitializeAsync() => _container.StartAsync();
    public Task DisposeAsync() => _container.DisposeAsync().AsTask();
}
```

Share via `ICollectionFixture` so one container handles many tests.

## JSON assertions

Deserialize, assert on fields. Don't string-match the response body.

```csharp
var response = await client.PostAsJsonAsync("/api/v1/orders", request);
response.StatusCode.Should().Be(HttpStatusCode.Created);

var order = await response.Content.ReadFromJsonAsync<OrderDto>();
order.Should().NotBeNull();
order!.CustomerName.Should().Be(request.CustomerName);
```

For `ProblemDetails`:

```csharp
var problem = await response.Content.ReadFromJsonAsync<ValidationProblemDetails>();
problem!.Errors.Should().ContainKey("CustomerName");
```

## Service unit tests

Plain unit tests, no HTTP, no host.

```csharp
public class OrderServiceTests
{
    [Fact]
    public async Task CreateAsync_RejectsInvalidProduct()
    {
        var catalog = Substitute.For<IProductCatalog>();
        catalog.ExistsAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
               .Returns(false);
        var repo = Substitute.For<IOrderRepository>();

        var service = new OrderService(catalog, repo);
        var result = await service.CreateAsync(new CreateOrderRequest(...), default);

        result.IsSuccess.Should().BeFalse();
        await repo.DidNotReceive().SaveAsync(Arg.Any<Order>(), Arg.Any<CancellationToken>());
    }
}
```

## Contract / snapshot tests

For published APIs, guard against accidental schema breaks:

```csharp
[Fact]
public async Task OpenApiSpec_IsStable()
{
    var client = _factory.CreateClient();
    var spec = await client.GetStringAsync("/openapi/v1.json");
    await Verifier.Verify(spec);  // Verify.Xunit
}
```

First run records the snapshot. Subsequent changes that modify the spec fail the test — a real signal that a contract changed.

## What to test

- Happy path per endpoint.
- Validation failure per required field.
- Auth: 401 when unauthenticated, 403 when wrong scope/role.
- Not found returns 404.
- Conflict / business rule returns 409 or 422 with correct shape.
- Rate limit kicks in after threshold.

## What not to test

- ASP.NET Core's own behavior (model binding mechanics, middleware order).
- Serializer defaults (System.Text.Json behavior).
- Framework validation attributes (`[Required]` works; don't test it).
- Exact JSON formatting — assert on deserialized objects.

## Speed

A good REST project's integration tests should run in under 30 seconds total. If they're slower, the culprit is usually:

- One database container per test (use a shared fixture).
- `WebApplicationFactory` rebuilt per test (use `IClassFixture`).
- Sleep-based waits (use polling with timeout).
