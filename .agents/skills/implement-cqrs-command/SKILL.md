---
name: implement-cqrs-command
description: Implements a new CQRS command following DDD patterns — command record, handler, validator, endpoint
---

## When to use
- When creating a new write operation (Create, Update, Delete)
- When adding a new use case that modifies state

## Command Template

```csharp
namespace Project.Application.Modules.{Module}.Commands;

public record Create{Entity}Command(
    // Command properties
) : ICommand
{
    public Guid CorrelationId() => Guid.NewGuid();
}
```

## Handler Template

```csharp
namespace Project.Application.Modules.{Module}.Commands;

public class Create{Entity}CommandHandler(
    ILogger<Create{Entity}CommandHandler> logger,
    I{Entity}Repository {entityVar}Repository)
: ICommandHandler<Create{Entity}Command>
{
    public async Task HandleAsync(Create{Entity}Command command, CancellationToken cancellationToken)
    {
        logger.LogInformation("Handling Create{Entity}Command");
        try
        {
            // 1. Validate business rules
            // 2. Build aggregate via factory method
            // 3. Persist
        }
        catch (Exception ex)
        {
            logger.LogError(ex, "Error handling Create{Entity}Command");
            throw;
        }
    }
}
```

## Validator Template

```csharp
namespace Project.Application.Modules.{Module}.Validators;

public class Create{Entity}CommandValidator : AbstractValidator<Create{Entity}Command>
{
    public Create{Entity}CommandValidator()
    {
        RuleFor(x => x.Name).NotEmpty();
        RuleFor(x => x.Email).EmailAddress();
    }
}
```

## CLI Commands

```bash
# Create module directories
mkdir -p src/Application/Modules/{Module}/Commands
mkdir -p src/Application/Modules/{Module}/Validators
mkdir -p src/Server/Endpoints

# Add NuGet packages (if not already added)
dotnet add package FluentValidation --project src/Application

# Verify compilation
dotnet build Project.slnx

# Run tests
dotnet test tests/Project.Application.Tests --filter "FullyQualifiedName~{Entity}"
```

## Steps

1. Create `record Command : ICommand` in `Application/Modules/{Module}/Commands/`
2. Create handler `: ICommandHandler<T>` in the same folder
3. Create (optional) the Validator in `Application/Modules/{Module}/Validators/`
4. Add endpoint in `Server/EndPoints/{Entity}CommandsEndpoints.cs`
5. Register the endpoint in `Server/Program.cs`
6. Verify compilation with `dotnet build Project.slnx`
