# Authentication and Authorization

## JWT bearer — the default

Stateless, scalable, standard. The API validates tokens issued by an external identity provider.

```csharp
builder.Services.AddAuthentication(JwtBearerDefaults.AuthenticationScheme)
    .AddJwtBearer(options =>
    {
        options.Authority = config["Auth:Authority"];   // e.g. https://login.example.com
        options.Audience = config["Auth:Audience"];     // this API's identifier
        options.RequireHttpsMetadata = !builder.Environment.IsDevelopment();
        options.TokenValidationParameters = new TokenValidationParameters
        {
            ValidateIssuer = true,
            ValidateAudience = true,
            ValidateLifetime = true,
            ValidateIssuerSigningKey = true,
            ClockSkew = TimeSpan.FromSeconds(30),
        };
    });
```

**Authority** — the OIDC provider's base URL. The middleware fetches `/.well-known/openid-configuration` automatically, then the JWKS endpoint for signing keys. Keys rotate; the middleware handles it.

**Audience** — the API's identifier in the IdP. Prevents a token minted for another API being accepted here.

**ClockSkew** — default is 5 minutes. Tighten to 30 seconds for production APIs. The default is generous for consumer-facing clocks; services have NTP.

## Authorization policies

Define named policies. Apply by name. Don't scatter `[Authorize(Roles="...")]` strings or raw `.RequireRole(...)` calls across endpoints.

```csharp
builder.Services.AddAuthorizationBuilder()
    .AddPolicy("orders:read", p => p.RequireClaim("scope", "orders:read"))
    .AddPolicy("orders:write", p => p.RequireClaim("scope", "orders:write"))
    .AddPolicy("admin", p => p.RequireRole("admin"));
```

Apply at the group level where possible:

```csharp
var group = app.MapGroup("/api/v1/orders")
    .RequireAuthorization("orders:read");  // default for the group

group.MapPost("/", Create).RequireAuthorization("orders:write");  // override
group.MapDelete("/{id:guid}", Delete).RequireAuthorization("admin");
```

### Scope vs role

- **Scopes** are capabilities granted to a client. `orders:read` says "this token can read orders."
- **Roles** are properties of the user. `admin` says "the user is an admin."

Scopes scale better for APIs — they describe what the token can do, independent of user identity. Use scopes for service-to-service and fine-grained capability design. Reserve roles for coarse user categorization.

## Requirement-based authorization

For rules that aren't single-claim checks (e.g., "can edit only own orders"):

```csharp
public sealed class OwnerRequirement : IAuthorizationRequirement;

public sealed class OwnerHandler(IOrderRepository repo)
    : AuthorizationHandler<OwnerRequirement, Order>
{
    protected override async Task HandleRequirementAsync(
        AuthorizationHandlerContext context,
        OwnerRequirement requirement,
        Order resource)
    {
        var userId = context.User.FindFirst("sub")?.Value;
        if (userId is not null && resource.OwnerId == userId)
        {
            context.Succeed(requirement);
        }
    }
}

services.AddSingleton<IAuthorizationHandler, OwnerHandler>();
services.AddAuthorizationBuilder()
    .AddPolicy("order-owner", p => p.AddRequirements(new OwnerRequirement()));
```

Invoke per-resource:

```csharp
var authResult = await authService.AuthorizeAsync(user, order, "order-owner");
if (!authResult.Succeeded) return TypedResults.Forbid();
```

## API keys (service-to-service)

For internal service calls that don't have a user identity, API keys are acceptable. Never for public APIs.

Authentication scheme:

```csharp
public sealed class ApiKeyAuthenticationHandler(
    IOptionsMonitor<ApiKeyOptions> options,
    ILoggerFactory logger,
    UrlEncoder encoder,
    IApiKeyStore store)
    : AuthenticationHandler<ApiKeyOptions>(options, logger, encoder)
{
    protected override async Task<AuthenticateResult> HandleAuthenticateAsync()
    {
        if (!Request.Headers.TryGetValue("X-Api-Key", out var header))
            return AuthenticateResult.NoResult();

        var key = header.ToString();
        var identity = await store.ValidateAsync(key, Context.RequestAborted);
        if (identity is null) return AuthenticateResult.Fail("Invalid API key");

        var claims = new[] { new Claim("sub", identity.ClientId) };
        var principal = new ClaimsPrincipal(new ClaimsIdentity(claims, Scheme.Name));
        return AuthenticateResult.Success(new AuthenticationTicket(principal, Scheme.Name));
    }
}
```

```csharp
builder.Services.AddAuthentication()
    .AddScheme<ApiKeyOptions, ApiKeyAuthenticationHandler>("ApiKey", _ => { });
```

Store keys hashed (bcrypt or argon2). Never compare plaintext. Rotate regularly.

## Getting user info in endpoints

```csharp
group.MapGet("/me", (ClaimsPrincipal user) =>
{
    var id = user.FindFirst("sub")?.Value;
    var email = user.FindFirst("email")?.Value;
    return TypedResults.Ok(new { Id = id, Email = email });
});
```

`ClaimsPrincipal` is bound automatically. Don't inject `IHttpContextAccessor` just to read the user.

## What not to do

- **Don't store passwords.** Ever. If you must authenticate users yourself, use ASP.NET Core Identity — don't roll password hashing.
- **Don't issue JWTs yourself** unless you're building an identity provider. Use an existing IdP.
- **Don't accept tokens over HTTP in production.** HTTPS only, enforced by `RequireHttpsMetadata` and HSTS.
- **Don't put sensitive data in JWT claims.** Tokens are visible to the client (they're signed, not encrypted).
- **Don't set long token lifetimes.** 15-60 minutes is standard. Use refresh tokens for longer sessions.
- **Don't invalidate JWTs by checking a database on every request.** Defeats the purpose of stateless auth. Short lifetimes + refresh tokens is the right answer.

## CORS

If the API is called from browsers on a different origin, configure CORS explicitly:

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("default", policy => policy
        .WithOrigins("https://app.example.com")
        .WithMethods("GET", "POST", "PUT", "DELETE")
        .WithHeaders("Authorization", "Content-Type")
        .WithExposedHeaders("X-Request-Id"));
});

app.UseCors("default");
```

Never `AllowAnyOrigin()` + credentials. It's rejected by browsers for good reason.
