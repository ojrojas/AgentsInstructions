---
name: create-value-object
description: Creates a ValueObject / StronglyTypedId in Domain/{Aggregate}/ — Kernel.Domain only, EF conversion lives in Infrastructure
---

## When to use
- When adding a new entity identifier (`StronglyTypedId`) or an immutable domain value with behavior

## Location (MANDATORY)

- Aggregate IDs and aggregate-owned values: `src/Services/{Service}/Domain/{Aggregate}/{Name}.cs`
  (e.g. `Domain/Users/UserId.cs`). Shared kernel values only: `Domain/Shared/`.
- Never `src/Core/Shared/` or `src/Core/Modules/{Module}/ValueObjects/` (retired layout).
- The EF conversion (`HasConversion(id => id.Value, v => new UserId(v))`) lives in
  `Infrastructure/Persistence/Configurations/{Entity}Configuration.cs` — NEVER in `Domain/`.

## Templates (Kernel.Domain — exact base types)

```csharp
namespace {Service}.Domain.{Aggregate};

public sealed record {Entity}Id(Guid Value) : StronglyTypedId<Guid>(Value)
{
    public static {Entity}Id New() => new(Guid.CreateVersion7());
    public static {Entity}Id From(Guid value) => new(value);
}
```

```csharp
namespace {Service}.Domain.{Aggregate};

public sealed class {Name} : ValueObject
{
    public {Name}(string value)
    {
        // CheckRule(new {Name}MustBeValidRule(value)); // invariants as rules when non-trivial
        Value = value;
    }

    public string Value { get; }

    protected override IEnumerable<object?> GetEqualityComponents()
    {
        yield return Value;
    }
}
```

## Rules

- Inherit `ValueObject` (equality via `GetEqualityComponents`) — `BaseValueObject` / `GetEquatibilityComponents` are legacy names and MUST NOT be used.
- IDs inherit `StronglyTypedId<T>` with `New()` (`Guid.CreateVersion7()`) + `From(...)` factories.
- `Domain/` MUST NOT reference `Microsoft.EntityFrameworkCore`, `IEventBus`, `ISender`, logging or HTTP. Conversions (`.HasConversion`, `.OwnsOne`) go to Infrastructure.
- Enums-as-concepts inherit `Enumeration<TEnum>`.

## Steps

1. Create the file under `Domain/{Aggregate}/` (or `Domain/Shared/` if truly cross-aggregate).
2. Add the `HasConversion` / `OwnsOne` mapping in the Infrastructure configuration.
3. Verify with `dotnet build <repo>.slnx`.
