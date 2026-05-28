---
name: create-value-object
description: Creates a strongly-typed Value Object following DDD patterns — with equality, factory methods, EF conversion
---

## When to use
- When adding a new entity identifier
- When encapsulating a domain value with behavior

## Template

```csharp
namespace Project.Core.Shared;  // or Core.Modules.{Module}.ValueObjects

public sealed class {Name}(Guid value) : BaseValueObject
{
    public Guid Value { get; private set; } = value;

    public static {Name} New() => new(Guid.CreateVersion7());
    public static {Name} From(Guid value) => new(value);

    protected override IEnumerable<object?> GetEquatibilityComponents()
    {
        yield return Value;
    }
}
```

## Variations
- **For Guid IDs**: Use `BaseValueObject` and `Guid.CreateVersion7()`
- **For record struct IDs**: Use `readonly record struct` for lightweight structs
- **For strings**: Use `string Value` property and primary constructor

## CLI Commands

```bash
# Create ValueObject directory
mkdir -p src/Core/Modules/{Module}/ValueObjects
mkdir -p src/Core/Shared

# Verify compilation
dotnet build Project.slnx
```

## Steps

1. Identify location: `Core/Shared/` (shared) or `Core/Modules/{Module}/ValueObjects/`
2. Decide between `class : BaseValueObject` or `readonly record struct`
3. Implement `GetEquatibilityComponents()`
4. Add `New()` factory method
5. Add conversion in EntityConfiguration (`.OwnsOne()` or `.HasConversion()`)
6. Verify compilation with `dotnet build Project.slnx`
