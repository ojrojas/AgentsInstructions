---
name: oro-libraries
description: >
  Mandatory skill for all .NET Core projects. Enforces use of the vendored
  BuildingBlocks (Kernel.Domain, CQRS, EventBus, EventBus.RabbitMQ,
  Kernel.Infrastructure, ServiceDefaults, Logger) copied from
  $HOME/Sources/BuildingBlocks into <repo>/src/BuildingBlocks and referenced
  via relative ProjectReference, plus Central Package Management (CPM) for
  external/test packages only.
paths:
  - "**/*.cs"
  - "**/*.csproj"
  - "**/*.slnx"
  - "**/*.props"
---

# BuildingBlocks (vendored) & Central Package Management

Canonical source of the libraries (never hardcode a concrete home path, always
use the variable):

```bash
$HOME/Sources/BuildingBlocks
```

Reference docs live in that repo (`README.md` at its root and
`examples/Identity/README.md`). This skill does NOT duplicate them; it defines
HOW to integrate the libraries into a new repo. When in doubt about behavior,
read the source under `$HOME/Sources/BuildingBlocks/src/`.

Available library projects (under `$HOME/Sources/BuildingBlocks/src/`):

- `BuildingBlocks.Kernel.Domain` — entities, aggregates, strongly-typed IDs, value objects, `Result`/`Error`, specifications.
- `BuildingBlocks.CQRS` — `ISender` dispatcher, handlers, pipeline behaviors, lightweight validation.
- `BuildingBlocks.EventBus` — `IntegrationEvent`, `IEventBus`, subscription manager.
- `BuildingBlocks.EventBus.RabbitMQ` — RabbitMQ bus over durable topic exchange.
- `BuildingBlocks.Kernel.Infrastructure` — `AppDbContextBase`, `EfRepository`, transactional outbox.
- `BuildingBlocks.ServiceDefaults` — OpenTelemetry, health checks, resilient HTTP, `IEndpoint` slices, `Result → HTTP`, `GlobalExceptionHandler`.
- `BuildingBlocks.Logger` — Serilog wiring (Console, File, Loki, Seq).

Multi-target of the libraries: `net10.0` and `net11.0`. Exception: Blazor WASM
hosts stay on `net10.0` only (see `examples/Identity` notes).

## 1. Vendoring (mandatory consumption model)

Do NOT consume these libraries from any NuGet feed. Do NOT add a
`nuget.config` source for them. The workflow is:

1. Copy the library sources into the destination repo:

   ```bash
   mkdir -p "<repo>/src/BuildingBlocks"
   cp -r "$HOME/Sources/BuildingBlocks/src/BuildingBlocks."* "<repo>/src/BuildingBlocks/"
   ```

2. Reference them with **relative `ProjectReference`** from the service
   projects (same pattern as `examples/Identity/Identity.Server.csproj`):

   ```xml
   <ItemGroup>
     <ProjectReference Include="..\BuildingBlocks\BuildingBlocks.CQRS\BuildingBlocks.CQRS.csproj" />
     <ProjectReference Include="..\BuildingBlocks\BuildingBlocks.EventBus\BuildingBlocks.EventBus.csproj" />
     <ProjectReference Include="..\BuildingBlocks\BuildingBlocks.EventBus.RabbitMQ\BuildingBlocks.EventBus.RabbitMQ.csproj" />
     <ProjectReference Include="..\BuildingBlocks\BuildingBlocks.Kernel.Domain\BuildingBlocks.Kernel.Domain.csproj" />
     <ProjectReference Include="..\BuildingBlocks\BuildingBlocks.Kernel.Infrastructure\BuildingBlocks.Kernel.Infrastructure.csproj" />
     <ProjectReference Include="..\BuildingBlocks\BuildingBlocks.ServiceDefaults\BuildingBlocks.ServiceDefaults.csproj" />
     <!-- Logger only when Serilog file/Loki/Seq sinks are needed -->
     <ProjectReference Include="..\BuildingBlocks\BuildingBlocks.Logger\BuildingBlocks.Logger.csproj" />
   </ItemGroup>
   ```

   Adjust the relative depth (`..`) to the real location of the consuming
   `.csproj`. Never use absolute paths and never hardcode `/home/oroja`.

3. Keep the vendored copy at `src/BuildingBlocks/` at the repo root (or under
   `src/` next to the services). Do not rename the library project folders.

## 2. Central Package Management (externals only)

`Directory.Packages.props` with `ManagePackageVersionsCentrally=true` is
mandatory, but ONLY for external/third-party packages (EF Core providers,
`RabbitMQ.Client`, OpenTelemetry, Serilog, OpenIddict, test packages, etc.)
and NEVER for the vendored BuildingBlocks (those are `ProjectReference`).

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <ItemGroup>
    <!-- Examples — pin real versions, never "*" -->
    <PackageVersion Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.9" />
    <PackageVersion Include="RabbitMQ.Client" Version="7.1.2" />
  </ItemGroup>
  <ItemGroup>
    <!-- Test packages live here too -->
    <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="18.4.0" />
    <PackageVersion Include="xunit" Version="2.9.3" />
    <PackageVersion Include="Moq" Version="4.20.72" />
    <PackageVersion Include="coverlet.collector" Version="10.0.0" />
  </ItemGroup>
</Project>
```

Rules:

- No `Version="*"` anywhere.
- No versions inside `.csproj` (`<PackageReference Include="xunit" />` only).
- Shared defaults (`Nullable enable`, `ImplicitUsings enable`, `LangVersion latest`)
  belong in `Directory.Build.props`.

## 3. BuildingBlocks.CQRS — dispatcher + Vertical Slice

Registration (order of behaviors matters — Logging first, Validation second):

```csharp
builder.Services.AddCqrs(cqrs => cqrs
    .RegisterHandlersFromAssemblyContaining<Program>()
    .AddOpenBehavior(typeof(LoggingBehavior<,>))
    .AddOpenBehavior(typeof(ValidationBehavior<,>)));
```

Notes:

- `AddCqrs` registers `ISender` → `Sender` (scoped) and
  `IDomainEventDispatcher` → `DomainEventDispatcher` (scoped), plus every
  `IRequestHandler<,>`, `IDomainEventHandler<>` and `IValidator<>` found in the
  given assemblies.
- Dispatch exclusively through `ISender.SendAsync<TResponse>(IRequest<TResponse>, ct)`.
  There is a single handler per request type.
- Organize each feature as a vertical slice in one folder/file: command or query
  record (`ICommand<Result<T>>` / `IQuery<Result<T>>`), optional
  `Validator<T>` with `RuleFor(...)`, handler
  (`ICommandHandler<TCommand, Result<T>>` / `IQueryHandler<TQuery, Result<T>>`),
  and endpoint (see §6).
- Domain events raised by aggregates are handled in-process via
  `IDomainEventHandler<TEvent>`; cross-service communication uses the outbox +
  EventBus (§5-§6), never domain event handlers directly.

## 4. Kernel.Domain — modelling rules

- Entities: inherit `Entity<TId>` (identity equality). Aggregates: inherit
  `AggregateRoot<TId>` and mutate only through factory methods / intention-named
  methods that call `CheckRule(new SomeRule(...))` and then
  `RaiseDomainEvent(new SomethingHappenedDomainEvent(...))`.
- IDs: declare `public sealed record OrderId(Guid Value) : StronglyTypedId<Guid>(Value);`
  and configure the EF conversion in the `DbContext`.
- Value objects: inherit `ValueObject`; enumerations: inherit
  `Enumeration<TEnum>`.
- Return `Result` / `Result<TValue>` from handlers (and domain factories where
  relevant) instead of throwing for expected failures. Build failures with
  `Error.Validation / .NotFound / .Conflict / .Unauthorized / .Forbidden / .Failure`.
- Queries: express them as `Specification<T>` subclasses calling `Where(...)`
  (accumulative AND), composed with `.And(...)` / `.Or(...)` / `.Not(...)`,
  plus `AddInclude`, ordering, paging, and `ApplyAsNoTracking()` for reads.
  Reuse `IsSatisfiedBy(entity)` in unit tests.

## 5. Kernel.Infrastructure — DbContext, repository, outbox

1. The service `DbContext` MUST inherit
   `AppDbContextBase(DbContextOptions, IDomainEventDispatcher)`. Its
   `SaveChangesAsync` drains aggregate domain events through the dispatcher
   before committing, so handler side-effects join the same transaction.
2. Register the context with EF (`AddDbContext<TDbContext>(o => o.UseNpgsql(...))`
   or the provider in use), then:

   ```csharp
   builder.Services.AddUnitOfWork<OrdersDbContext>();
   builder.Services.AddOutbox<OrdersDbContext>();
   ```

   `AddUnitOfWork` exposes the already-registered context as `IUnitOfWork`.
   `AddOutbox` registers `IOutboxWriter` (scoped) and the background
   `OutboxProcessor<TDbContext>`.
3. Include the outbox entity in the model:

   ```csharp
   protected override void OnModelCreating(ModelBuilder modelBuilder)
   {
       modelBuilder.ApplyConfiguration(new OutboxEntityTypeConfiguration());
       // ... rest of the model
   }
   ```

4. Flow inside a command handler: load/create aggregate via
   `IRepository<TAggregate, TId>` (`EfRepository`, specification-capable:
   `GetByIdAsync`, `AddAsync`, `FirstOrDefaultAsync(spec)`, `ListAsync(spec)`,
   `AnyAsync`/`CountAsync`), call `IOutboxWriter.StageAsync(new
   SomethingIntegrationEvent(...), ct)` for cross-service facts, then a single
   `IUnitOfWork.SaveChangesAsync(ct)`. Never publish to `IEventBus` directly
   from the handler — the outbox processor does it after commit.

## 6. ServiceDefaults + EventBus wiring in `Program.cs`

Canonical order:

```csharp
var builder = WebApplication.CreateBuilder(args);

builder.AddServiceDefaults(); // OTel (OTLP only if OTEL_EXPORTER_OTLP_ENDPOINT is set) + /health + resilient HttpClients

builder.Services.AddCqrs(cqrs => cqrs
    .RegisterHandlersFromAssemblyContaining<Program>()
    .AddOpenBehavior(typeof(LoggingBehavior<,>))
    .AddOpenBehavior(typeof(ValidationBehavior<,>)));

builder.Services.AddDbContext<OrdersDbContext>(o => o.UseNpgsql(connString));
builder.Services.AddUnitOfWork<OrdersDbContext>();
builder.Services.AddOutbox<OrdersDbContext>();

builder.Services
    .AddRabbitMqEventBus(builder.Configuration)
    .AddSubscription<OrderCreatedIntegrationEvent, OrderCreatedHandler>();

builder.Services.AddEndpoints(typeof(Program).Assembly);
builder.Services.AddExceptionHandler<GlobalExceptionHandler>();
builder.Services.AddProblemDetails();

var app = builder.Build();
app.UseExceptionHandler();
app.MapDefaultEndpoints(); // GET /health (all checks) + GET /alive (live tag)
app.MapEndpoints();        // all IEndpoint slices
app.Run();
```

Endpoint slice pattern — implement `IEndpoint` per feature and return domain
results mapped to HTTP:

```csharp
public sealed class CreateOrderEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapPost("/orders", async (CreateOrderCommand command, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.SendAsync(command, ct);
            return result.ToCreatedResult(id => $"/orders/{id}");
        });
}
```

`ToHttpResult()` → `204 NoContent` / ProblemDetails; `ToHttpResult<T>()` →
`200 Ok` / ProblemDetails; `ToCreatedResult(...)` → `201 Created` / ProblemDetails
(`ErrorType` decides the status code). `GlobalExceptionHandler` + `UseExceptionHandler()`
is the only global error path; do not add custom exception middleware.

EventBus configuration (`appsettings.json`):

```json
{
  "EventBus": {
    "RabbitMq": {
      "HostName": "localhost",
      "UserName": "guest",
      "Password": "guest",
      "ExchangeName": "integration_events",
      "QueueName": "orders-service"
    }
  }
}
```

Each service owns its `QueueName`; publishing goes to the durable topic
`ExchangeName` with confirms, consuming via a `BackgroundService` with manual
ack and exponential retries. Delivery is at-least-once: integration handlers
MUST be idempotent. Local broker: `docker run -d -p 5672:5672 -p 15672:15672 rabbitmq:4-management`.

## 7. Logger (only when needed)

ServiceDefaults already wires OpenTelemetry logging. Add `BuildingBlocks.Logger`
only for Serilog sinks (Console/File/Loki/Seq):

```csharp
// Host startup:
builder.Host.UseBuildingBlocksLogger();
// Non-host scenarios (tests, libraries):
services.AddBuildingBlocksLogger(configuration);
```

Options bind from the `LoggerOptions` configuration section, with an optional
`Action<LoggerOptions>` override.

## 8. Checklist + forbidden legacy

Starting or reviewing a solution:

- [ ] Libraries vendored at `<repo>/src/BuildingBlocks/` from `$HOME/Sources/BuildingBlocks`, referenced via relative `ProjectReference`.
- [ ] No `nuget.config` feed for BuildingBlocks; CPM only for externals/tests with pinned versions.
- [ ] `AddCqrs` with assembly scan + Logging/Validation behaviors; `SendAsync` only.
- [ ] Aggregates with `StronglyTypedId`, rules, domain events; handlers return `Result`.
- [ ] `AppDbContextBase` + `AddUnitOfWork` + `AddOutbox` + `OutboxEntityTypeConfiguration`; handlers stage outbox then single `SaveChangesAsync`.
- [ ] `AddServiceDefaults` + `AddRabbitMqEventBus(...).AddSubscription` + `AddEndpoints` + `GlobalExceptionHandler` + `MapDefaultEndpoints/MapEndpoints`.
- [ ] `EventBus:RabbitMq` section present; integration handlers idempotent.

PROHIBITED (legacy `Oro*` world — fail review if found):

- `https://nuget.pkg.github.com/ojrojas`, `OroCQRS`, `OroKernel.Shared`, `OroServiceDefaults`, `OroEventBus`, `OroEventBusRabbitMQ`, `OroLoggers`.
- `AddCqrsHandlers()`, `ISender.Send(...)` (non-Async), `BaseEntity`, `AuditableDbContext`.
- Absolute home paths such as `/home/oroja/...` — always `$HOME/...`.
- `Version="*"` or versions hardcoded in `.csproj`.

Migration note for old repos: delete the GitHub Packages `nuget.config` source
and the `Oro*` `PackageVersion`/`PackageReference` entries, vendor the sources
per §1, then apply §3-§7 replacing namespaces (`OroCQRS.Core.*` →
`BuildingBlocks.CQRS.*`, `OroKernel.Shared.*` → `BuildingBlocks.Kernel.*`,
`OroBuildingBlocks.*` → `BuildingBlocks.*`) and the registration sequence in §6.
