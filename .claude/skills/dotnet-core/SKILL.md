# .NET Core Backend Skill (C# 14 / .NET 10)

## Core Rules

You must use modern .NET 10 and C# 14 features by default.

## Mandatory Language Features

Always prefer:

### 1. C# 14 features
- Extension Members (preferred over static helper classes)
- `field` keyword for property validation
- Primary constructors for services and records
- Collection expressions
- Improved pattern matching
- `Span<T>` / `ReadOnlySpan<T>` for allocations
- File-scoped namespaces

### 2. Architecture style
- Minimal APIs preferred over controllers (unless complexity requires MVC)
- Vertical slice architecture when possible
- Dependency injection via constructor only
- Avoid heavy abstractions

### 3. Performance rules
- Avoid allocations in hot paths
- Prefer `ValueTask` when appropriate
- Use streaming (`IAsyncEnumerable<T>`) for large datasets

### 4. API design
- Explicit contracts (DTOs)
- No over-engineered layers
- Use `record` for request/response models

### 5. Observability
- Structured logging (ILogger)
- OpenTelemetry tracing recommended
- Correlation IDs required

### 6. Error handling
- Use ProblemDetails for APIs
- Never swallow exceptions silently