---
name: create-new-module
description: Creates a complete DDD module — aggregate, value objects, repository, CQRS, endpoints, DI registration
---

## When to use
- When introducing a new domain concept into the system
- When a new entity requires full CRUD operations

## Structure to create

```
src/Core/Modules/{Module}/
├── Aggregates/{Entity}.cs
├── DomainEvents/{Entity}CreateEvent.cs
├── ValueObjects/{Entity}Id.cs          (if needed)
├── Repositories/I{Entity}Repository.cs

src/Application/Modules/{Module}/
├── Commands/
│   ├── Create{Entity}Command.cs
│   ├── Create{Entity}CommandHandler.cs
│   └── Delete{Entity}Command.cs
├── Queries/
│   ├── Get{Entity}ByIdQuery.cs
│   └── Get{Entity}sQuery.cs
└── DTOs/{Entity}Dto.cs

src/Infrastructure/
├── Data/EntityConfigurations/{Entity}EntityConfiguration.cs
├── Repositories/{Entity}Repository/{Entity}Repository.cs

src/Server/EndPoints/
├── {Entity}CommandsEndpoints.cs
└── {Entity}QueriesEndpoints.cs
```

## CLI Commands

```bash
# Create project files (if module needs separate project)
dotnet new classlib -n Project.Core.Modules.{Module} -o src/Core/Modules/{Module}
dotnet new classlib -n Project.Application.Modules.{Module} -o src/Application/Modules/{Module}

# Add project references
dotnet add src/Application/Modules/{Module}/{Module}.csproj reference src/Core/Modules/{Module}/{Module}.csproj

# Build and verify
dotnet build Project.slnx
```

## Steps

1. **Core**: Create Aggregate, Value Objects, Domain Events, Repository Interface
2. **Application**: Create Commands, Queries, Handlers, DTOs
3. **Infrastructure**: Create EF Configuration, Repository Implementation
4. **Server**: Create Endpoints, register in Program.cs
5. **DI**: Register repository in infrastructure extension class
6. Run `dotnet build Project.slnx` and verify
