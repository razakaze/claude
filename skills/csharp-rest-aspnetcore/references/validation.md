# Validation

## Built-in (.NET 10)

`builder.Services.AddValidation()` plus data annotations on request types. A source generator emits the validation logic at compile time. No runtime reflection, no per-endpoint boilerplate.

```csharp
public sealed record CreateOrderRequest(
    [Required, MinLength(1)] string CustomerName,
    [Required, EmailAddress] string Email,
    [Required, MinLength(1)] IReadOnlyList<OrderLineRequest> Lines);

public sealed record OrderLineRequest(
    [Required] Guid ProductId,
    [Range(1, 1000)] int Quantity,
    [Range(typeof(decimal), "0.01", "1000000")] decimal UnitPrice);
```

Invalid input returns 400 with:

```json
{
  "type": "https://tools.ietf.org/html/rfc9110#section-15.5.1",
  "title": "One or more validation errors occurred.",
  "status": 400,
  "errors": {
    "CustomerName": ["The CustomerName field is required."],
    "Lines[0].Quantity": ["The field Quantity must be between 1 and 1000."]
  }
}
```

Your endpoint doesn't run. No `ModelState` check needed.

## Common annotations

| Attribute | Purpose |
|---|---|
| `[Required]` | Non-null, non-empty string |
| `[MinLength(n)]` / `[MaxLength(n)]` | Collection/string length |
| `[StringLength(max, MinimumLength=min)]` | Combined string length |
| `[Range(min, max)]` | Numeric range |
| `[EmailAddress]` | RFC-style email format |
| `[Url]` | Well-formed URL |
| `[RegularExpression(pattern)]` | Regex |
| `[Compare(nameof(OtherProp))]` | Equality to another property (e.g., confirm password) |

## Cross-field validation

Data annotations don't support cross-field rules cleanly. Two options:

### Option A: IValidatableObject

```csharp
public sealed class DateRangeRequest : IValidatableObject
{
    [Required] public DateOnly Start { get; init; }
    [Required] public DateOnly End { get; init; }

    public IEnumerable<ValidationResult> Validate(ValidationContext context)
    {
        if (End < Start)
            yield return new ValidationResult("End must be on or after Start.",
                [nameof(End), nameof(Start)]);
    }
}
```

Works with the built-in validation pipeline. Good for rules on one request type.

### Option B: FluentValidation (for heavy validation logic)

When rules are complex, async, or need to be reused across request types, FluentValidation is still the best tool.

```xml
<PackageReference Include="FluentValidation" Version="12.*" />
<PackageReference Include="FluentValidation.DependencyInjectionExtensions" Version="12.*" />
```

```csharp
public sealed class CreateOrderValidator : AbstractValidator<CreateOrderRequest>
{
    public CreateOrderValidator(IProductCatalog catalog)
    {
        RuleFor(x => x.CustomerName).NotEmpty();
        RuleFor(x => x.Email).NotEmpty().EmailAddress();
        RuleForEach(x => x.Lines).ChildRules(line =>
        {
            line.RuleFor(l => l.Quantity).InclusiveBetween(1, 1000);
            line.RuleFor(l => l.ProductId).MustAsync(async (id, ct) =>
                await catalog.ExistsAsync(id, ct)).WithMessage("Unknown product");
        });
    }
}
```

Wire via an endpoint filter:

```csharp
public sealed class ValidationFilter<T> : IEndpointFilter
{
    public async ValueTask<object?> InvokeAsync(
        EndpointFilterInvocationContext context, EndpointFilterDelegate next)
    {
        var validator = context.HttpContext.RequestServices.GetService<IValidator<T>>();
        if (validator is null) return await next(context);

        var arg = context.Arguments.OfType<T>().FirstOrDefault();
        if (arg is null) return await next(context);

        var result = await validator.ValidateAsync(arg, context.HttpContext.RequestAborted);
        if (!result.IsValid)
        {
            var errors = result.Errors
                .GroupBy(e => e.PropertyName)
                .ToDictionary(g => g.Key, g => g.Select(e => e.ErrorMessage).ToArray());
            return TypedResults.ValidationProblem(errors);
        }

        return await next(context);
    }
}

group.MapPost("/", Create).AddEndpointFilter<ValidationFilter<CreateOrderRequest>>();
```

Don't mix built-in validation and FluentValidation on the same type — pick one per request DTO.

## Rule of thumb

- **Data annotations** for simple, declarative rules. 90% of cases.
- **`IValidatableObject`** for the cross-field rule that doesn't justify FluentValidation.
- **FluentValidation** when rules depend on async calls, external services, or are complex enough that annotations bury them.

## What not to validate in the request

- **Authorization** — that's auth middleware's job. If user can't see order X, that's 403, not validation.
- **Referential integrity with the database** (e.g., "does this category exist"). Move to the service layer; validation should be cheap and about request shape.
- **Business invariants** that change based on system state. These belong in the domain model.

Validation confirms the request is well-formed. Not that the request will succeed.
