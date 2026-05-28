---
name: oro-libraries
description: >
  Mandatory skill for all .NET Core projects. Enforces use of the Oro
  ecosystem (OroCQRS, OroBuildingBlocks, OroKernel) from the GitHub Packages
  NuGet feed at nuget.pkg.github.com/ojrojas, and Central Package Management
  (CPM) via Directory.Packages.props.
paths:
  - "**/*.cs"
  - "**/*.csproj"
  - "**/*.slnx"
  - "**/*.props"
---

# Oro Libraries & Central Package Management

## 1. NuGet Source Configuration

All .NET Core projects MUST reference the Oro packages from the GitHub
Packages NuGet feed. Create or update `nuget.config` at the repository root:

```xml
<?xml version="1.0" encoding="utf-8"?>
<configuration>
  <packageSources>
    <clear />
    <add key="nuget.org" value="https://api.nuget.org/v3/index.json" />
    <add key="github-ojrojas" value="https://nuget.pkg.github.com/ojrojas/index.json" />
  </packageSources>
  <packageSourceCredentials>
    <github-ojrojas>
      <add key="Username" value="ojrojas" />
      <add key="ClearTextPassword" value="%GITHUB_PACKAGES_TOKEN%" />
    </github-ojrojas>
  </packageSourceCredentials>
</configuration>
```

The `GITHUB_PACKAGES_TOKEN` environment variable must contain a GitHub PAT
with `read:packages` scope.

## 2. Central Package Management (CPM) — Mandatory

Every repository with multiple .NET projects MUST use CPM. Single-project
repos SHOULD also use it for consistency.

### 2.1 Directory.Packages.props

Create `Directory.Packages.props` at the repository root:

```xml
<Project>
  <PropertyGroup>
    <ManagePackageVersionsCentrally>true</ManagePackageVersionsCentrally>
  </PropertyGroup>
  <!-- Oro packages (always pinned) -->
  <ItemGroup>
    <PackageVersion Include="OroCQRS" Version="1.0.0" />
    <PackageVersion Include="OroKernel.Shared" Version="1.0.1" />
    <PackageVersion Include="OroServiceDefaults" Version="*" />
    <PackageVersion Include="OroLoggers" Version="*" />
    <PackageVersion Include="OroEventBus" Version="*" />
    <PackageVersion Include="OroEventBusRabbitMQ" Version="*" />
  </ItemGroup>
</Project>
```

Replace `Version="*"` with the actual latest version once published.

### 2.2 Directory.Build.props — Central Imports & Defaults

```xml
<Project>
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
  </PropertyGroup>
</Project>
```

### 2.3 Referencing packages in .csproj (no version — comes from CPM)

```xml
<ItemGroup>
  <PackageReference Include="OroCQRS" />
  <PackageReference Include="OroKernel.Shared" />
  <PackageReference Include="OroServiceDefaults" />
  <PackageReference Include="OroEventBusRabbitMQ" />
</ItemGroup>
```

## 3. OroCQRS — Lightweight CQRS/Mediator (Package: `OroCQRS`)

Replaces MediatR. Targets `net10.0`, requires `Microsoft.AspNetCore.App`.

### 3.1 NuGet Package

| Field | Value |
|---|---|
| PackageId | `OroCQRS` |
| Version | `1.0.0` |
| Namespace (root) | `OroCQRS.Core` |

### 3.2 Message Interfaces

```csharp
using OroCQRS.Core.Interfaces;

// Marker — base of all messages
public interface IBaseMessage
{
    Guid CorrelationId();
}

// Requests (with or without result)
public interface IRequest : IBaseMessage;
public interface IRequest<out IResult> : IBaseMessage;

// Commands (void or with result)
public interface ICommand : IRequest;
public interface ICommand<out TResult> : IRequest<TResult>;

// Queries (always return result)
public interface IQuery<out TResult> : IRequest<TResult>;

// Notifications (void or with result)
public interface INotification : IRequest;
public interface INotification<out TResult> : IRequest<TResult>;
```

### 3.3 Handler Interfaces

```csharp
// Command Handlers
public interface ICommandHandler<in TCommand> where TCommand : ICommand
{
    Task HandleAsync(TCommand command, CancellationToken cancellationToken);
}
public interface ICommandHandler<in TCommand, TResult> where TCommand : ICommand<TResult>
{
    Task<TResult> HandleAsync(TCommand command, CancellationToken cancellationToken);
}

// Query Handlers
public interface IQueryHandler<in TQuery, TResult> where TQuery : IQuery<TResult>
{
    Task<TResult> HandleAsync(TQuery query, CancellationToken cancellationToken);
}

// Notification Handlers
public interface INotificationHandler<in TNotification> where TNotification : INotification
{
    Task HandleAsync(TNotification notification, CancellationToken cancellationToken);
}
public interface INotificationHandler<in TNotification, TResult> where TNotification : INotification<TResult>
{
    Task<TResult> HandleAsync(TNotification notification, CancellationToken cancellationToken);
}
```

### 3.4 Mediator (Sender)

```csharp
public interface ISender
{
    Task Send(ICommand request, CancellationToken ct);
    Task<TResult> Send<TResult>(ICommand<TResult> request, CancellationToken ct);
    Task Send(INotification request, CancellationToken ct);
    Task<TResult> Send<TResult>(INotification<TResult> request, CancellationToken ct);
    Task<TResult> Send<TResult>(IQuery<TResult> request, CancellationToken ct);
    Task<TResult?> Send<TResult>(object request, CancellationToken ct);
}
```

### 3.5 Registration

```csharp
using OroCQRS.Core.Extensions;

// Scans calling assemblies and registers all handlers as Scoped
// Also registers ISender -> Sender as Scoped
builder.Services.AddCqrsHandlers();
// Or scan specific assemblies:
builder.Services.AddCqrsHandlers(typeof(MyCommandHandler).Assembly);
```

### 3.6 Usage Patterns

**Command (void):**
```csharp
public record CreateUserCommand(string UserName) : ICommand
{
    public Guid CorrelationId() => Guid.NewGuid();
}

public class CreateUserCommandHandler : ICommandHandler<CreateUserCommand>
{
    public async Task HandleAsync(CreateUserCommand command, CancellationToken ct)
    {
        // Business logic
    }
}

// Dispatch:
await sender.Send(new CreateUserCommand("oscar"), ct);
```

**Query (with result):**
```csharp
public record GetUserQuery(Guid UserId) : IQuery<User>
{
    public Guid CorrelationId() => Guid.NewGuid();
}

public class GetUserQueryHandler : IQueryHandler<GetUserQuery, User>
{
    public async Task<User> HandleAsync(GetUserQuery query, CancellationToken ct)
    {
        return await dbContext.Users.FindAsync(query.UserId);
    }
}

// Dispatch:
var user = await sender.Send(new GetUserQuery(userId), ct);
```

**Notification:**
```csharp
public record UserCreatedNotification(string Email) : INotification
{
    public Guid CorrelationId() => Guid.NewGuid();
}

public class SendWelcomeEmailHandler : INotificationHandler<UserCreatedNotification>
{
    public async Task HandleAsync(UserCreatedNotification notification, CancellationToken ct)
    {
        // Send email
    }
}

// Dispatch:
await sender.Send(new UserCreatedNotification("user@example.com"), ct);
```

## 4. OroBuildingBlocks — Microservice Infrastructure

### 4.1 OroServiceDefaults (Package: `OroServiceDefaults`)

```csharp
using OroBuildingBlocks.ServiceDefaults;

// In Program.cs:
builder.AddServiceDefaults();
  // Registers: OpenTelemetry (logs, metrics, tracing, OTLP)
  //            Health checks (/health, /alive)
  //            Service Discovery
  //            HTTP client resilience (standard)

app.MapDefaultEndpoints();
  // Maps GET /health (all checks), GET /alive (live tag)
  // Only in Development environment
```

Additional utilities:
```csharp
// ClaimsPrincipal extensions
User.GetUserId()       // reads "sub" claim
User.GetUserName()     // reads ClaimTypes.Name

// Configuration extensions
config.GetRequiredValue("Key")  // throws InvalidOperationException if missing

// Data Protection (supports File, Redis, AzureBlob)
services.AddConfiguredDataProtection(config, env);

// Identity endpoints (OpenIddict)
app.MapIdentityEndpoints(options => { ... });
```

### 4.2 OroLoggers (Package: `OroLoggers`)

```csharp
using OroBuildingBlocks.Loggers;

// Create a Serilog logger with Console + Seq sinks
var logger = LoggerPrinter.CreateSerilogLogger("AppName", "ServiceName", configuration);

// Or use Aspire integration
builder.AddServicesWritersLogger(config);
// Reads ConnectionStrings:Seq for Seq endpoint
```

### 4.3 OroEventBus (Package: `OroEventBus`)

```csharp
using OroBuildingBlocks.EventBus.Abstractions;
using OroBuildingBlocks.EventBus.Events;

// Base event
public record IntegrationEvent
{
    public Guid Id { get; }          // auto-generated
    public DateTime Created { get; } // auto-generated UtcNow
}

// Abstraction
public interface IEventBus
{
    Task PublishAsync(IntegrationEvent integrationEvent, CancellationToken ct = default);
}

// Handler
public interface IIntegrationEventHandler<in TEvent> where TEvent : IntegrationEvent
{
    Task Handle(TEvent integrationEvent);
}
```

### 4.4 OroEventBusRabbitMQ (Package: `OroEventBusRabbitMQ`)

```csharp
using OroBuildingBlocks.EventBusRabbitMQ;

// Registration (uses Aspire RabbitMQ integration)
builder.AddRabbitMqEventBus("connectionName")
       .AddSubscriptionManager<OrderSubmittedEvent, OrderSubmittedHandler>()
       .ConfigureJsonOptions(opts => { /* custom JSON settings */ });

// Configuration (appsettings.json):
{
  "ConnectionStrings": {
    "connectionName": "amqp://localhost"
  },
  "EventBus": {
    "SubscriptionClientName": "MyServiceQueue",
    "RetryCount": 5
  }
}
```

## 5. OroKernel — Shared Kernel (Package: `OroKernel.Shared`)

Requires `OroCQRS` as a dependency (domain events extend INotification).

### 5.1 Base Entities

```csharp
using OroKernel.Shared.Entities;

// GUID-based entity (uses Guid.CreateVersion7())
public abstract class BaseEntity : WithDomainEventBase
{
    public Guid Id { get; set; }
}

// Typed ID entity
public abstract class BaseEntity<TId> : WithDomainEventBase
    where TId : struct, IEquatable<TId>
{
    public TId Id { get; set; }
}

// Value Object
public abstract class BaseValueObject : IEquatable<BaseValueObject>
{
    protected abstract IEnumerable<object?> GetEquatibilityComponents();
}
```

### 5.2 Domain Events

```csharp
using OroKernel.Shared.Events;
using OroKernel.Shared.Interfaces;

// Base domain event record (extends INotification from OroCQRS)
public abstract record DomainEventBase : IDomainEvent
{
    public DateTime OcurredOn { get; } = DateTime.UtcNow;
    public Guid CorrelationId() => Guid.NewGuid();
}

// Raise from entity
public class Order : BaseEntity
{
    public void Submit()
    {
        RaiseDomainEvent(new OrderSubmittedEvent(Id));
    }
}
```

### 5.3 Auditable DbContext

```csharp
using OroKernel.Shared.Data;
using OroKernel.Shared.Interfaces;

public class MyDbContext : AuditableDbContext
{
    public MyDbContext(DbContextOptions options, IUserInfoProvider userInfoProvider)
        : base(options, userInfoProvider) { }

    public DbSet<Product> Products => Set<Product>();
}
```

### 5.4 Repository Pattern

```csharp
using OroKernel.Shared.Interfaces;

public interface IRepository<T> : IRepositoryBase<T> where T : class, IAggregateRoot
{
    Task<T?> GetByIdAsync<TId>(TId id, CancellationToken ct);
}

public interface IRepositoryBase<T> where T : class, IAggregateRoot
{
    Task AddAsync(T entity, CancellationToken ct);
    Task UpdateAsync(T entity, CancellationToken ct);
    Task DeleteAsync(T entity, CancellationToken ct);
    Task<IEnumerable<T>> GetAllAsync(CancellationToken ct);
    Task<IEnumerable<T>> FindAsync(Expression<Func<T, bool>> predicate, CancellationToken ct);
    Task<T?> FindSingleAsync(Expression<Func<T, bool>> predicate, CancellationToken ct);
}
```

### 5.5 Identity / User Resolution

```csharp
using OroKernel.Shared.Options;
using OroKernel.Shared.Services;
using OroKernel.Shared.Interfaces;

// Register in DI:
services.Configure<UserInfo>(opts =>
{
    opts.Id = Guid.Empty;
    opts.UserName = "System";
    opts.Email = "system@example.com";
});
services.AddTransient<IPostConfigureOptions<UserInfo>, ClaimsUserInfoService>();
services.AddScoped<IUserInfoProvider, DefaultUserInfoProvider>();

// Optional: HttpClient for external identity server
services.AddTransient<RetryDelegatingHandler>();
services.AddHttpClient<IIdentityClientService, IdentityClientService>((sp, client) =>
{
    client.BaseAddress = new Uri("https://identity.example/");
    client.Timeout = TimeSpan.FromSeconds(10);
})
.AddHttpMessageHandler<RetryDelegatingHandler>();
```

### 5.6 Specification Pattern

```csharp
using OroKernel.Shared.Specification;

public class ActiveUsersSpecification : BaseSpecification<User>
{
    public override Expression<Func<User, bool>> ToExpression()
        => user => user.IsActive;
}

// Combinators
var spec = new ActiveUsersSpecification()
    .And(new UsersFromCountrySpecification("CO"))
    .Or(new VIPUsersSpecification());
```

### 5.7 Domain Exceptions

```csharp
using OroKernel.Shared.Exceptions;

throw new DomainException("ORDER_INVALID", "Order cannot be processed");
```

## 6. CPM Enforcement for Test Projects

Test projects MUST also use CPM. Add test package versions to the central
`Directory.Packages.props`:

```xml
<ItemGroup>
  <!-- Test packages -->
  <PackageVersion Include="Microsoft.NET.Test.Sdk" Version="18.4.0" />
  <PackageVersion Include="xunit" Version="2.9.3" />
  <PackageVersion Include="xunit.runner.visualstudio" Version="3.1.5" />
  <PackageVersion Include="Moq" Version="4.20.72" />
  <PackageVersion Include="Microsoft.EntityFrameworkCore.InMemory" Version="10.0.7" />
  <PackageVersion Include="coverlet.collector" Version="10.0.0" />
</ItemGroup>
```

Test .csproj files then only contain:

```xml
<Project Sdk="Microsoft.NET.Sdk">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <IsPackable>false</IsPackable>
    <IsTestProject>true</IsTestProject>
  </PropertyGroup>
  <ItemGroup>
    <PackageReference Include="Microsoft.NET.Test.Sdk" />
    <PackageReference Include="xunit" />
    <PackageReference Include="Moq" />
    <PackageReference Include="coverlet.collector" />
  </ItemGroup>
</Project>
```

## 7. Complete Project Setup Checklist

When starting a new .NET Core solution:

- [ ] Create `nuget.config` with GitHub Packages source + credentials
- [ ] Create `Directory.Packages.props` with `ManagePackageVersionsCentrally`
- [ ] Add Oro package versions and test package versions to CPM
- [ ] Create `Directory.Build.props` with shared settings
- [ ] Reference Oro packages WITHOUT versions in .csproj
- [ ] Register OroCQRS handlers via `builder.Services.AddCqrsHandlers()`
- [ ] Add OroBuildingBlocks defaults via `builder.AddServiceDefaults()` and `app.MapDefaultEndpoints()`
- [ ] Configure AuditableDbContext with IUserInfoProvider for auditing
