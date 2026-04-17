---
name: csharp-rest-aspnetcore
description: Use when writing or modifying REST APIs in ASP.NET Core on C# 14 / .NET 10, using Minimal API by default with Controllers only when specific features require them. Triggers on mentions of WebApplication, MapGet/MapPost/MapPut/MapDelete, TypedResults, ProblemDetails, AddValidation, endpoint filters, JWT bearer authentication, OpenAPI/Swagger, API versioning, rate limiting, IHttpClientFactory, or building HTTP services in .NET. Use whenever the user is adding an endpoint, wiring authentication, designing error responses, versioning an API, or choosing between Minimal API and Controllers. Consult even when the user doesn't name these terms — any .NET project with Microsoft.NET.Sdk.Web or a WebApplication.CreateBuilder call is in scope.
---

# C# REST API with ASP.NET Core (.NET 10 / C# 14)

Build REST APIs with **Minimal API** as the default. Switch to **Controllers** only when specific features require them (listed below). Use `TypedResults` for typed return values. Validate with built-in `AddValidation()` (new in .NET 10). Errors as `ProblemDetails` (RFC 7807). JWT bearer for auth. OpenAPI via the built-in `Microsoft.AspNetCore.OpenApi`.

---

## Minimal API vs Controllers — the rule

**Default: Minimal API.** Since .NET 10 added built-in validation, the historical reasons to prefer Controllers have mostly evaporated.

**Use Controllers when**, and only when:

1. **OData.** The OData library is Controller-first; Minimal API support is second-class.
2. **Model binding you can't express with parameter-level `[FromBody]` / `[FromQuery]` / `[FromRoute]` / `[AsParameters]`.** Rare. `[AsParameters]` handles almost all of it.
3. **Complex filter pipelines** where filters need to inspect model state, set response types based on action metadata, or interact with an action-descriptor model. Endpoint filters in Minimal API are simpler but less introspective.
4. **Legacy team convention.** If the codebase is all Controllers, don't mix — consistency beats the switch cost.

For everything else — CRUD, microservices, internal APIs, gateways — Minimal API.

**Don't mix in the same project** unless there's a reason. One WebApplication can host both, but the mental overhead of "which style lives where" isn't worth it for consistency-by-default.

---

## Project shape

### csproj

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <TreatWarningsAsErrors>true</TreatWarningsAsErrors>
    <InvariantGlobalization>true</InvariantGlobalization>
    <!-- Required for Minimal API validation source generator -->
    <InterceptorsNamespaces>$(InterceptorsNamespaces);Microsoft.AspNetCore.Http.Validation.Generated</InterceptorsNamespaces>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="Microsoft.AspNetCore.OpenApi" Version="10.0.*" />
    <PackageReference Include="Microsoft.AspNetCore.Authentication.JwtBearer" Version="10.0.*" />
    <PackageReference Include="Asp.Versioning.Http" Version="9.*" />
  </ItemGroup>
</Project>
```

`InvariantGlobalization` shrinks the deployment and is the right default for APIs that don't format locale-sensitive output.

### Program.cs

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.Services.AddOpenApi();
builder.Services.AddValidation();
builder.Services.AddProblemDetails();

builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = builder.Configuration["Auth:Authority"];
        options.Audience = builder.Configuration["Auth:Audience"];
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ClockSkew = TimeSpan.FromSeconds(30),
        };
    });
builder.Services.AddAuthorization();

builder.Services.AddRateLimiter(options =>
{
    options.AddFixedWindowLimiter("default", o =>
    {
        o.PermitLimit = 100;
        o.Window = TimeSpan.FromMinutes(1);
    });
});

builder.Services.AddHttpClient<IExternalApi, ExternalApi>(client =>
{
    client.BaseAddress = new Uri(builder.Configuration["ExternalApi:BaseUrl"]!);
    client.Timeout = TimeSpan.FromSeconds(30);
});

builder.Services.AddSingleton<IOrderRepository, OrderRepository>();

var app = builder.Build();

app.UseExceptionHandler();
app.UseStatusCodePages();

if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
}

app.UseAuthentication();
app.UseAuthorization();
app.UseRateLimiter();

app.MapOrdersApi();   // extension method, see below
app.MapHealthChecks("/health");

app.Run();
```

### Rules

- **`UseExceptionHandler()` with `AddProblemDetails()`** converts unhandled exceptions to RFC 7807 responses. Without it, exceptions leak stack traces in dev and give opaque 500s in prod.
- **Middleware order matters.** Exception handler first, auth before authorization, rate limiter after auth (so anonymous requests are limited by a different key).
- **Register `HttpClient` through `IHttpClientFactory`** — never `new HttpClient()`. Socket exhaustion is real.

---

## Endpoint organization

Don't write 300 endpoints in `Program.cs`. Group by resource in extension methods.

```csharp
public static class OrdersEndpoints
{
    public static IEndpointRouteBuilder MapOrdersApi(this IEndpointRouteBuilder app)
    {
        var group = app.MapGroup("/api/v1/orders")
            .RequireAuthorization()
            .RequireRateLimiting("default")
            .WithTags("Orders");

        group.MapGet("/", GetAll).WithName("GetAllOrders");
        group.MapGet("/{id:guid}", GetById).WithName("GetOrderById");
        group.MapPost("/", Create).WithName("CreateOrder");
        group.MapPut("/{id:guid}", Update).WithName("UpdateOrder");
        group.MapDelete("/{id:guid}", Delete).WithName("DeleteOrder");

        return app;
    }

    private static async Task<Ok<PagedResult<OrderDto>>> GetAll(
        [AsParameters] OrderQuery query,
        IOrderRepository repository,
        CancellationToken ct)
    {
        var result = await repository.GetAllAsync(query, ct);
        return TypedResults.Ok(result);
    }

    private static async Task<Results<Ok<OrderDto>, NotFound>> GetById(
        Guid id,
        IOrderRepository repository,
        CancellationToken ct)
    {
        var order = await repository.GetByIdAsync(id, ct);
        return order is null ? TypedResults.NotFound() : TypedResults.Ok(order);
    }

    private static async Task<Results<Created<OrderDto>, ValidationProblem>> Create(
        CreateOrderRequest request,
        IOrderRepository repository,
        CancellationToken ct)
    {
        var created = await repository.CreateAsync(request, ct);
        return TypedResults.Created($"/api/v1/orders/{created.Id}", created);
    }
    // ...
}
```

### Rules

- **`TypedResults`, not `Results`.** Typed returns participate in OpenAPI generation automatically — you get the right schema without `.Produces<T>()` calls.
- **`Results<T1, T2, ...>` union type** for endpoints that can return multiple shapes. OpenAPI reflects every variant.
- **`[AsParameters]`** to bind a group of related query params to a struct. Cleaner than N individual parameters.
- **`CancellationToken` on every async endpoint.** ASP.NET Core binds it to the request abort token automatically.
- **`MapGroup` for shared concerns** — auth, rate limiting, tags, route prefix. Stack them; don't repeat on every endpoint.
- **`.WithName(...)`** on each endpoint. Used by OpenAPI (operationId) and `LinkGenerator`.
- **No logic in the endpoint delegate.** Endpoints dispatch to services. If you're writing a loop or a branch in the delegate, extract.

---

## Request/response contracts

Use `record`s for request and response DTOs. Validation via data annotations.

```csharp
public sealed record CreateOrderRequest(
    [Required, MinLength(1)] string CustomerName,
    [Required, MinLength(1)] IReadOnlyList<OrderLineRequest> Lines);

public sealed record OrderLineRequest(
    [Required] Guid ProductId,
    [Range(1, 1000)] int Quantity);

public sealed record OrderDto(
    Guid Id,
    string CustomerName,
    decimal Total,
    IReadOnlyList<OrderLineDto> Lines);
```

With `AddValidation()` registered, invalid requests return `400` with a `ValidationProblemDetails` payload automatically. No manual `ModelState.IsValid` checks.

### Rules

- **Separate request and response DTOs.** Don't expose entities. Don't reuse a DTO as both input and output.
- **`IReadOnlyList<T>`** for collection properties in DTOs — preserves immutability semantics.
- **`required` members** where appropriate. Compile-time enforcement is stronger than `[Required]`, but only checks construction, not deserialization. Use both if the input truly must have a value.
- **No circular references in DTOs.** Flatten.
- **No domain logic on DTOs.** They're data contracts. Business logic lives in services.

See `references/validation.md` for cross-field rules, async validators, and FluentValidation integration when annotations aren't enough.

---

## Error handling

All errors go out as `ProblemDetails`. Three paths:

1. **Validation errors** — automatic via `AddValidation()`. Returns 400 with `ValidationProblemDetails`.
2. **Expected failures** (not found, conflict) — return `TypedResults.NotFound()`, `TypedResults.Conflict()`, etc.
3. **Unexpected exceptions** — caught by `UseExceptionHandler()` and converted to `ProblemDetails`.

Enrich the problem with trace info:

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        context.ProblemDetails.Extensions["traceId"] = context.HttpContext.TraceIdentifier;
        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;
    };
});
```

### Rules

- **Don't throw for expected flows.** A missing resource is `NotFound`, not `throw new KeyNotFoundException`. Exceptions are expensive and clutter logs.
- **Don't return raw strings on error.** `return BadRequest("invalid")` is useless to clients. Use `TypedResults.Problem(...)` with structured detail.
- **Never leak internal exception messages** in production. The exception handler suppresses stack traces by default in non-Development — keep it that way.
- **422 Unprocessable Entity** for semantic validation failures that aren't 400 (e.g., business rule violations on otherwise-valid input). Distinguish from 400 "malformed request."

See `references/error-handling.md` for custom exception handlers, retry-after headers, and status code selection matrix.

---

## Authentication and authorization

JWT bearer is the default. The identity provider issues tokens; the API validates them.

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = config["Auth:Authority"];
        options.Audience = config["Auth:Audience"];
        options.RequireHttpsMetadata = !builder.Environment.IsDevelopment();
    });

builder.Services.AddAuthorizationBuilder()
    .AddPolicy("admin", p => p.RequireRole("admin"))
    .AddPolicy("orders:write", p => p.RequireClaim("scope", "orders:write"));
```

Apply via route group or per-endpoint:

```csharp
group.RequireAuthorization("orders:write");

// or per endpoint
group.MapDelete("/{id:guid}", Delete).RequireAuthorization("admin");
```

### Rules

- **Authority + Audience, not shared secret.** Use an OIDC provider (IdentityServer, Azure AD, Auth0, Keycloak) and validate against its JWKS.
- **Require HTTPS in production** (`RequireHttpsMetadata = true`). Only relax in dev.
- **Policies, not inline `[Authorize(Roles=...)]` strings** scattered across endpoints. Named policies are testable and reusable.
- **Never validate JWTs by hand.** The middleware does it correctly — signature, issuer, audience, expiry, clock skew.

See `references/auth.md` for scope design, API key authentication for service-to-service, and refresh token patterns.

---

## Versioning

```csharp
builder.Services.AddApiVersioning(options =>
{
    options.DefaultApiVersion = new ApiVersion(1, 0);
    options.AssumeDefaultVersionWhenUnspecified = true;
    options.ReportApiVersions = true;
}).AddApiExplorer(options =>
{
    options.GroupNameFormat = "'v'VVV";
});

var v1 = app.NewVersionedApi().MapGroup("/api/v{version:apiVersion}");
v1.MapOrdersApi().HasApiVersion(1, 0);
```

### Rules

- **URL path versioning** (`/api/v1/...`) is the default. Clearest, cacheable, unambiguous.
- **Don't break a version.** If a change breaks contract, release v2. The API explorer groups endpoints per version for separate OpenAPI docs.
- **Deprecate, don't delete.** Mark old versions deprecated (`.IsApiVersionNeutral()` / `.HasDeprecatedApiVersion(...)`) and remove after a published sunset date.

---

## OpenAPI

`AddOpenApi()` + `MapOpenApi()` produces the spec at `/openapi/v1.json` by default. For Swagger UI or Scalar, bolt on one package — but don't expose the UI in production unless you mean to.

```csharp
if (app.Environment.IsDevelopment())
{
    app.MapOpenApi();
    app.MapScalarApiReference();  // needs Scalar.AspNetCore package
}
```

Annotate endpoints for documentation without repeating yourself:

```csharp
group.MapPost("/", Create)
    .WithSummary("Create a new order")
    .WithDescription("Creates an order for the authenticated user. Idempotency key supported via header.")
    .Produces<OrderDto>(StatusCodes.Status201Created)
    .ProducesValidationProblem();
```

`TypedResults` handles most of `.Produces` automatically — add it only when the return type doesn't tell the full story.

---

## Outbound HTTP

Always `IHttpClientFactory`. Never `new HttpClient()`. Named or typed clients; typed preferred for strong coupling to a specific API.

```csharp
public sealed class ExternalApi(HttpClient http) : IExternalApi
{
    public async Task<ExternalResource> FetchAsync(Guid id, CancellationToken ct)
    {
        var response = await http.GetFromJsonAsync<ExternalResource>(
            $"resources/{id}", ct);
        return response ?? throw new InvalidOperationException("null response");
    }
}
```

```csharp
builder.Services.AddHttpClient<IExternalApi, ExternalApi>(client =>
{
    client.BaseAddress = new Uri(config["ExternalApi:BaseUrl"]!);
    client.Timeout = TimeSpan.FromSeconds(30);
});
```

Add Polly for retry/circuit-breaker:

```csharp
builder.Services.AddHttpClient<IExternalApi, ExternalApi>(...)
    .AddStandardResilienceHandler();
```

`AddStandardResilienceHandler` (from `Microsoft.Extensions.Http.Resilience`) gives you retry, timeout, rate limiter, and circuit breaker with sane defaults. Tune if measurements say so, not because you might need it later.

---

## Testing

Integration tests with `WebApplicationFactory<Program>`. Unit tests for services.

```csharp
public class OrdersEndpointsTests(WebApplicationFactory<Program> factory) : IClassFixture<WebApplicationFactory<Program>>
{
    [Fact]
    public async Task GetById_ReturnsOrder_WhenExists()
    {
        var client = factory.WithWebHostBuilder(builder =>
        {
            builder.ConfigureServices(services =>
            {
                var stub = Substitute.For<IOrderRepository>();
                stub.GetByIdAsync(Arg.Any<Guid>(), Arg.Any<CancellationToken>())
                    .Returns(new OrderDto(Guid.NewGuid(), "Alice", 99m, []));
                services.Replace(ServiceDescriptor.Singleton(stub));
            });
        }).CreateClient();

        var response = await client.GetAsync("/api/v1/orders/11111111-1111-1111-1111-111111111111");

        response.StatusCode.Should().Be(HttpStatusCode.OK);
        var order = await response.Content.ReadFromJsonAsync<OrderDto>();
        order!.CustomerName.Should().Be("Alice");
    }
}
```

`Program.cs` must be partial for the factory to find it:

```csharp
public partial class Program { }
```

See `references/testing.md` for auth stubbing, database fixtures, and snapshot testing.

---

## C# 14 idioms worth using here

- **Primary constructors** on service classes for DI.
- **Collection expressions** (`[]`) for empty collections in DTOs and defaults.
- **`required` members** on DTOs where the compile-time check adds real safety.
- **Pattern matching on `Results<>` unions** when inspecting results in tests.

---

## What this skill does not cover

- **SignalR / WebSockets.** Real-time features need a different shape; not covered here.
- **gRPC.** Related but uses `Grpc.AspNetCore` and has its own patterns.
- **GraphQL.** Different library stack (HotChocolate); not covered.
- **Server-side rendering / Razor Pages / Blazor.** This skill is HTTP JSON APIs only.
- **Complex OData query pipelines.** OData is Controller-first; reach for Controllers per the decision rule above.
