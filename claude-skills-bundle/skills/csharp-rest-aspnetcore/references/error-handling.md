# Error Handling

## Layers

1. **Validation** — 400 with `ValidationProblemDetails`, automatic via `AddValidation()`.
2. **Expected flow errors** — `TypedResults.NotFound()`, `Conflict()`, `BadRequest(...)` returning `ProblemDetails`.
3. **Unexpected exceptions** — caught by `UseExceptionHandler()`, converted to 500 `ProblemDetails`.

## Status code matrix

| Condition | Status | Typed Result |
|---|---|---|
| Resource doesn't exist | 404 | `TypedResults.NotFound()` |
| Authenticated but forbidden | 403 | Automatic via policy |
| Not authenticated | 401 | Automatic via bearer |
| Malformed request | 400 | Automatic via validation |
| Semantic validation failure (e.g., business rule) | 422 | `TypedResults.UnprocessableEntity(problem)` |
| Duplicate / conflict | 409 | `TypedResults.Conflict(problem)` |
| Rate limited | 429 | Automatic via rate limiter |
| Internal error | 500 | `TypedResults.Problem()` or exception handler |
| Service unavailable (dep down) | 503 | `TypedResults.Problem(..., statusCode: 503)` |

**400 vs 422:** 400 means "I can't parse this." 422 means "I parsed it, but the content violates a rule." If your validator fires, it's 400. If the business logic rejects valid-shaped input, it's 422.

## Centralized ProblemDetails

```csharp
builder.Services.AddProblemDetails(options =>
{
    options.CustomizeProblemDetails = context =>
    {
        var httpContext = context.HttpContext;
        context.ProblemDetails.Extensions["traceId"] = httpContext.TraceIdentifier;
        context.ProblemDetails.Extensions["timestamp"] = DateTime.UtcNow;

        if (httpContext.User.Identity?.IsAuthenticated ?? false)
        {
            context.ProblemDetails.Extensions["requestId"] = httpContext.Connection.Id;
        }
    };
});
```

Every error response carries a `traceId` — correlate with logs, don't make the caller describe the problem in text.

## Custom exception handler

For unhandled exceptions, register an `IExceptionHandler` implementation — cleaner than middleware:

```csharp
public sealed class GlobalExceptionHandler(
    IProblemDetailsService problemDetails,
    ILogger<GlobalExceptionHandler> logger) : IExceptionHandler
{
    public async ValueTask<bool> TryHandleAsync(
        HttpContext context,
        Exception exception,
        CancellationToken cancellationToken)
    {
        logger.LogError(exception,
            "Unhandled exception. TraceId: {TraceId}",
            context.TraceIdentifier);

        context.Response.StatusCode = exception switch
        {
            ArgumentException => StatusCodes.Status400BadRequest,
            UnauthorizedAccessException => StatusCodes.Status403Forbidden,
            _ => StatusCodes.Status500InternalServerError,
        };

        return await problemDetails.TryWriteAsync(new ProblemDetailsContext
        {
            HttpContext = context,
            Exception = exception,
            ProblemDetails = new ProblemDetails
            {
                Type = $"https://httpstatuses.io/{context.Response.StatusCode}",
                Status = context.Response.StatusCode,
                Title = GetTitle(exception),
            },
        });
    }

    private static string GetTitle(Exception ex) => ex switch
    {
        ArgumentException => "Invalid argument",
        UnauthorizedAccessException => "Access denied",
        _ => "Internal server error",
    };
}

builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
```

### Rules

- **Never include the exception message in the response in production.** Stack traces leak implementation details. Log them; don't send them.
- **Map specific exception types to specific status codes.** A domain `NotFoundException` becomes 404, not 500.
- **Don't throw for control flow.** Repository returns `null`? Don't throw `NotFoundException` in the service; return a nullable or a `Result<T>` union.

## Result pattern (alternative to exceptions)

For services that can fail predictably, a `Result<T>` discriminated union beats exceptions:

```csharp
public readonly record struct Result<T>(T? Value, string? Error)
{
    public static Result<T> Success(T value) => new(value, null);
    public static Result<T> Failure(string error) => new(default, error);
    public bool IsSuccess => Error is null;
}
```

Or use a library (`OneOf`, `LanguageExt`, `FluentResults`). Pick one per codebase.

Endpoints then translate to HTTP:

```csharp
var result = await service.DoThingAsync(input, ct);
return result.IsSuccess
    ? TypedResults.Ok(result.Value)
    : TypedResults.Problem(result.Error, statusCode: 422);
```

Exceptions stay for the truly exceptional: DB unreachable, invariant violated, programmer error.

## Retry-After

For 429 and 503, set the header:

```csharp
return TypedResults.Problem(
    "Too many requests",
    statusCode: StatusCodes.Status429TooManyRequests,
    extensions: new Dictionary<string, object?>
    {
        ["retryAfter"] = 30,
    }).AddHeader("Retry-After", "30");
```

Well-behaved clients back off. Badly-behaved clients get rate-limited harder; that's their problem.

## Logging in the handler

Log once, at the exception boundary. Don't log-and-rethrow — every layer logs, you get 5x duplicates, and diagnosing which is the original is a nightmare.

```csharp
// BAD
try { ... }
catch (Exception ex)
{
    logger.LogError(ex, "Failed");
    throw;
}

// GOOD — let the exception handler log
try { ... }
catch (SpecificRecoverableException ex)
{
    logger.LogWarning(ex, "Transient failure, retrying");
    // retry or return fallback
}
```

Wrapping exceptions for translation is fine. Logging at every level is not.
