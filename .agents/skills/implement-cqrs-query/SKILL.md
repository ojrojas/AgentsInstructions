---
name: implement-cqrs-query
description: Implements a new CQRS query following DDD patterns — query record, handler, response DTO, endpoint
---

## When to use
- When creating a new read operation (Get, List, Search)
- When exposing data through endpoints

## Query Template

```csharp
namespace Project.Application.Modules.{Module}.Queries;

public record Get{Entity}ByIdQuery(Guid Id) : IQuery<Get{Entity}ByIdQueryResponse>
{
    public Guid CorrelationId() => Guid.NewGuid();
}
```

## Handler Template

```csharp
namespace Project.Application.Modules.{Module}.Queries;

public class Get{Entity}ByIdQueryHandler(
    ILogger<Get{Entity}ByIdQueryHandler> logger,
    I{Entity}Repository repository)
    : IQueryHandler<Get{Entity}ByIdQuery, Get{Entity}ByIdQueryResponse>
{
    public async Task<Get{Entity}ByIdQueryResponse> HandleAsync(
        Get{Entity}ByIdQuery query, CancellationToken cancellationToken)
    {
        Get{Entity}ByIdQueryResponse response = new();
        logger.LogInformation("Handling Get{Entity}ByIdQuery with Id: {Id}", query.Id);
        response.Data = await repository.Get{Entity}ByIdAsync(new(query.Id), cancellationToken);
        logger.LogInformation("Successfully handled Get{Entity}ByIdQuery");
        return response;
    }
}
```

## Response DTO Template

```csharp
namespace Project.Application.Modules.{Module}.DTOs;

public record Get{Entity}ByIdQueryResponse
{
    public {Entity}Dto? Data { get; set; }
}
```

## CLI Commands

```bash
# Create module directories
mkdir -p src/Application/Modules/{Module}/Queries
mkdir -p src/Application/Modules/{Module}/DTOs
mkdir -p src/Server/Endpoints

# Verify compilation
dotnet build Project.slnx

# Run tests
dotnet test tests/Project.Application.Tests --filter "FullyQualifiedName~{Entity}"
```

## Steps

1. Create `record Query : IQuery<TResponse>` in `Application/Modules/{Module}/Queries/`
2. Create handler `: IQueryHandler<T, TResult>` in the same folder
3. Create the response DTO
4. Add endpoint in `Server/EndPoints/{Entity}QueriesEndpoints.cs`
5. Register endpoint in `Server/Program.cs`
6. Verify compilation with `dotnet build Project.slnx`
