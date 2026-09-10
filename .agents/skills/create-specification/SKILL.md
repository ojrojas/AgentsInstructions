---
name: create-specification
description: Creates a Specification for reusable queries following the project pattern
---

## When to use
- When you need a specific and reusable query
- To encapsulate complex search criteria

## Template

```csharp
namespace Project.Infrastructure.Specifications;

public class Get{Entity}By{Criteria}Specification(
    {ParameterType} {parameterName}) : ISpecification<{Entity}>
{
    public Expression<Func<{Entity}, bool>> Criteria { get; } =
        entity => entity.{Property} == {parameterName};
}
```

## CLI Commands

```bash
# Create directory
mkdir -p src/Infrastructure/Specifications

# Verify compilation
dotnet build Project.slnx
```

## Steps

1. Create in `Infrastructure/Specifications/`
2. Implement `ISpecification<T>`
3. Use in repository with `FirstOrDefaultAsync(spec.Criteria, ...)`
