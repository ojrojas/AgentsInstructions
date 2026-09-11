---
name: implement-cqrs-command
description: Implements a write slice (command + validator + handler + endpoint) in one file under Application/Features/{Context}/ — BuildingBlocks Result, ISender, outbox
---

## When to use
- When creating a new write operation (Create, Update, Delete) for an existing aggregate
- New aggregates start with `create-new-module`, then one slice per use-case with this skill

## Slice location (MANDATORY)

`src/Services/{Service}/Application/Features/{Module}/Create{Entity}.cs` — command, validator,
handler and endpoint live in ONE file named after the feature. Never
`Application/Modules/{Module}/Commands|Validators/` + `Server/Endpoints/` splits.

Canon: `examples/Identity/Identity.Server/Application/Features/Users/RegisterUser.cs`.

## Slice template

```csharp
namespace {Service}.Application.Features.{Module};

public sealed record Create{Entity}Command(/* props */) : ICommand<Result<Guid>>;

public sealed class Create{Entity}Validator : Validator<Create{Entity}Command>
{
    public Create{Entity}Validator()
    {
        RuleFor(x => !string.IsNullOrWhiteSpace(x.Name), nameof(Create{Entity}Command.Name), "El nombre es obligatorio.");
    }
}

public sealed class Create{Entity}Handler(
    I{Entity}Repository repository,
    IUnitOfWork unitOfWork,
    IOutboxWriter outbox) : ICommandHandler<Create{Entity}Command, Result<Guid>>
{
    public async Task<Result<Guid>> HandleAsync(Create{Entity}Command command, CancellationToken ct)
    {
        // 1. Specification check (e.g. AnyAsync(spec)) → Error.Conflict / .Validation
        // 2. Aggregate factory (CheckRule + RaiseDomainEvent inside domain)
        // 3. await repository.AddAsync(aggregate, ct);
        // 4. await outbox.StageAsync(new {Entity}CreatedIntegrationEvent(...), ct); // when cross-service
        // 5. await unitOfWork.SaveChangesAsync(ct); // dispatches domain events + outbox atomically
        // 6. return aggregate.Id.Value;
    }
}

public sealed class Create{Entity}Endpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapPost("/{entities}", async (Create{Entity}Command command, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.SendAsync(command, ct);
            return result.ToCreatedResult(id => $"/{entities}/{id}");
        });
}
```

## Rules

- Dispatch only via `ISender.SendAsync`; return `Result`/`Result<T>` (`Error.Validation/.NotFound/.Conflict`), never throw for expected failures.
- Never call `IEventBus` from the handler — stage via `IOutboxWriter.StageAsync`, single `SaveChangesAsync`.
- Never touch `DbContext` directly in the slice; go through `IRepository<T,TId>` (`GetByIdAsync`, `AddAsync`, `FirstOrDefaultAsync(spec)`, `ListAsync(spec)`, `AnyAsync`, `CountAsync`).
- HTTP mapping only with `ToHttpResult()` / `ToCreatedResult(...)`; no custom exception middleware (`GlobalExceptionHandler` only).

## Steps

1. Create `Application/Features/{Module}/Create{Entity}.cs` with the four types above.
2. Endpoints are auto-discovered via `AddEndpoints` + `MapEndpoints` — no manual `Program.cs` endpoint registration beyond the standard wiring.
3. Verify with `dotnet build <repo>.slnx`; tests: `tests/Services/{Service}/Application/Features/{Module}/` (validator + handler `Result` paths; integration POST round-trip + outbox).
