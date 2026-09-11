---
name: create-specification
description: Creates a Specification<T> next to its aggregate in Domain/{Aggregate}/Specifications/ — composable Where, reused by handlers and unit tests
---

## When to use
- When you need a reusable business query (lookup, uniqueness check, search criteria)

## Location (MANDATORY)

`src/Services/{Service}/Domain/{Aggregate}/Specifications/{Entity}Specifications.cs` — next to the
aggregate it queries (canon: `Domain/Users/Specifications/UserSpecifications.cs`).
Never `src/Infrastructure/Specifications/` (that puts domain language in Infrastructure).

## Template (Kernel.Domain — exact API)

```csharp
namespace {Service}.Domain.{Aggregate}.Specifications;

public static class {Entity}Specifications
{
    public static Specification<{Entity}> ById({Entity}Id id) =>
        new Specification<{Entity}>().Where(e => e.Id == id).ApplyAsNoTracking();

    public static Specification<{Entity}> WithTakenIdentifier(string email, string userName) =>
        new Specification<{Entity}>().Where(e => e.Email == email || e.UserName == userName);
}
```

Compose with `.And(...)` / `.Or(...)` / `.Not(...)`, plus `AddInclude`, ordering, paging.
`Where(...)` calls accumulate with AND.

## Rules

- Specifications are DOMAIN language: they live in `Domain/`, reference only the aggregate and its value objects. No EF (`IQueryable`, `Include` chains), no DTOs, no HTTP.
- Handlers consume them via `IRepository<T,TId>` (`FirstOrDefaultAsync(spec)`, `ListAsync(spec)`, `AnyAsync(spec)`, `CountAsync(spec)`).
- Unit tests reuse them via `spec.IsSatisfiedBy(entity)` — never duplicate the predicate in tests.

## Steps

1. Create/extend `Domain/{Aggregate}/Specifications/{Entity}Specifications.cs`.
2. Use from the slice handler through the repository (no direct `DbContext`).
3. Verify with `dotnet build <repo>.slnx`.
