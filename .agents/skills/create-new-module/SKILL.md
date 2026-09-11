---
name: create-new-module
description: Creates a new aggregate + vertical slice in the canonical single-service layout (Domain/{Aggregate}/ + Application/Features/{Context}/ + Infrastructure/) — BuildingBlocks, Result, outbox
---

## When to use
- When introducing a new domain concept (new aggregate) into an existing service
- When adding the first slice for a new aggregate (later slices use `implement-cqrs-command` / `implement-cqrs-query`)

## Canonical layout (MANDATORY — mirrors `examples/Identity/Identity.Server`)

Single service = single `.csproj`. Never create `src/Core`, `src/Application`,
`src/Infrastructure`, `src/Server` classlibs, never `Modules/{X}/application,domain,persistence`
siblings, never top-level `Commands/`, `Handlers/`, `Queries/`, `Endpoints/` layer folders.

```text
src/Services/{Service}/
  Domain/{Entity}/
    {Entity}.cs                        # : AggregateRoot<{Entity}Id>
    {Entity}Id.cs                      # : StronglyTypedId<Guid>
    Events/{Entity}CreatedDomainEvent.cs
    Rules/                             # IBusinessRule implementations
    Specifications/{Entity}Specifications.cs
    I{Entity}Repository.cs             # interface ONLY
  Application/Features/{Module}/
    Create{Entity}.cs                  # slice: Command + Validator + Handler + Endpoint (one file)
    Get{Entity}ById.cs                 # slice: Query + Handler + Endpoint (+ Response)
  Application/IntegrationEvents/{Entity}CreatedIntegrationEvent.cs
  Application/DomainEventHandlers/{Entity}CreatedDomainEventHandler.cs  # StageAsync → outbox
  Infrastructure/
    Persistence/{Service}DbContext.cs  # : AppDbContextBase (ONLY DbContext location)
    Persistence/Configurations/{Entity}Configuration.cs  # IEntityTypeConfiguration<T>
    {Entity}Repository.cs              # : EfRepository<{Entity},{Entity}Id>
```

## Steps

1. **Domain** (`Domain/{Entity}/` — pure, no EF / no bus / no HTTP):
   aggregate with private ctor + intention-named factory (`CheckRule` + `RaiseDomainEvent`),
   `StronglyTypedId`, `Specification<T>` with `Where(...)`, repository interface.
2. **Slice** (`Application/Features/{Module}/Create{Entity}.cs` — whole feature in one file):
   `record Create{Entity}Command(...) : ICommand<Result<Guid>>`, `Validator<Create{Entity}Command>`
   with `RuleFor(...)`, `ICommandHandler<Create{Entity}Command, Result<Guid>>` loading via
   `IRepository`, staging integration events with `IOutboxWriter.StageAsync` + single
   `IUnitOfWork.SaveChangesAsync(ct)`, `IEndpoint` mapping via `ISender.SendAsync` +
   `ToCreatedResult` / `ToHttpResult`.
3. **Integration**: `IntegrationEvent` + `IDomainEventHandler<T>` that stages the outbox
   (never `IEventBus` directly from the command handler); register context with
   `AddUnitOfWork` + `AddOutbox`, `OutboxEntityTypeConfiguration` in `OnModelCreating`.
4. **Infrastructure** (`Infrastructure/` ONLY): `EfRepository` impl, `IEntityTypeConfiguration`
   (StronglyTypedId `HasConversion`), migrations under `Persistence/Migrations/`.
5. Run `dotnet build <repo>.slnx` and verify. Tests mirror the slice:
   `tests/Services/{Service}/Application/Features/{Module}/`.

## Forbidden (fail review)

- `src/Core/Modules/{Module}/`, `src/Application/Modules/{Module}/Commands|Queries|DTOs`,
  `src/Infrastructure/Data/...`, `src/Server/EndPoints/` — retired Clean-Architecture layout.
- Any `Persistence/` folder outside `Infrastructure/`, any EF type inside `Domain/`.
- Handler publishing via `IEventBus` instead of `IOutboxWriter.StageAsync`.
