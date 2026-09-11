---
name: implement-cqrs-query
description: Implements a read slice (query + handler + endpoint + response DTO) in one file under Application/Features/{Context}/ — Specification reads, NoTracking
---

## When to use
- When creating a new read operation (Get, List, Search) for an existing aggregate

## Slice location (MANDATORY)

`src/Services/{Service}/Application/Features/{Module}/Get{Entity}ById.cs` — query, handler,
endpoint and response live in ONE file named after the feature. Never
`Application/Modules/{Module}/Queries|DTOs/` + `Server/EndPoints/` splits.

Canon: `examples/Identity/Identity.Server/Application/Features/Users/GetCurrentUser.cs`.

## Slice template

```csharp
namespace {Service}.Application.Features.{Module};

public sealed record Get{Entity}ByIdQuery(Guid Id) : IQuery<Result<Get{Entity}ByIdResponse>>;

public sealed record Get{Entity}ByIdResponse({Entity}Dto Data);

public sealed class Get{Entity}ByIdHandler(
    I{Entity}Repository repository) : IQueryHandler<Get{Entity}ByIdQuery, Result<Get{Entity}ByIdResponse>>
{
    public async Task<Result<Get{Entity}ByIdResponse>> HandleAsync(Get{Entity}ByIdQuery query, CancellationToken ct)
    {
        // var entity = await repository.FirstOrDefaultAsync(new {Entity}ByIdSpec(query.Id), ct);
        // if (entity is null) return Error.NotFound(...);
        // return new Get{Entity}ByIdResponse(entity.ToDto());
    }
}

public sealed class Get{Entity}ByIdEndpoint : IEndpoint
{
    public void MapEndpoint(IEndpointRouteBuilder app) =>
        app.MapGet("/{entities}/{id:guid}", async (Guid id, ISender sender, CancellationToken ct) =>
        {
            var result = await sender.SendAsync(new Get{Entity}ByIdQuery(id), ct);
            return result.ToHttpResult();
        });
}
```

## Rules

- Reads go through `Specification<T>` (`Where(...)` + `ApplyAsNoTracking()`), reused via `IsSatisfiedBy` in unit tests. No raw LINQ with `Include` chains in the handler — push it into the specification.
- Return `Result<T>` (`Error.NotFound` when missing); map with `ToHttpResult()`.
- No `IOutboxWriter` / `SaveChangesAsync` in queries (reads are side-effect free).

## Steps

1. Create `Application/Features/{Module}/Get{Entity}ById.cs` (or `List{Entities}.cs` for collections with paging spec).
2. Endpoints auto-discovered via `AddEndpoints` + `MapEndpoints`.
3. Verify with `dotnet build <repo>.slnx`; tests mirror the slice (spec `IsSatisfiedBy`, handler NotFound/Ok paths, GET round-trip).
