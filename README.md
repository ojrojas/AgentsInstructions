# AgentsInstructions

Agents + skills para desarrollo profesional de software. Diseñado para opencode, compatible con otros runtimes que soporten agentes universales (markdown agent defs + subagent delegation + file tools).

## Agents

| Agent | Mode | Rol |
|---|---|---|
| `orchestrator` | primary | Coordinación, question tool, fases de ejecución |
| `planner` | subagent | Planes de implementación (WHAT, no HOW) |
| `coder` | subagent | Implementa código, sigue convenciones del repo |
| `tester` | subagent | Ejecuta tests, Gate: PASS o BLOCKED |

### Flujo

```
Orchestrator → Planner (plan) → Coder (código) → Tester (tests) → Reporte
```

- Solo el Orchestrator usa el question tool
- Subagentes no editan archivos fuera de su scope
- Un task termina solo cuando Coder + Tester reportan PASS

## Skills

### Auto-load (activación por archivos)

| Skill | Trigger | Descripción |
|---|---|---|
| `oro-libraries` | `**/*.cs`, `**/*.csproj`, `**/*.slnx`, `**/*.props` | BuildingBlocks vendored + CPM (mandatory .NET) |
| `dotnet-core` | `**/*.cs`, `**/*.csproj` | Convenciones .NET Core, C# moderno |
| `ngrx-signal-store` | `**/*.ts`, `**/*.store.ts` | NgRx SignalStore para Angular |
| `efcore-patterns` | `**/*DbContext.cs`, `**/*Configuration.cs` | EF Core best practices |
| `exception-handling` | `**/Program.cs`, `**/*ExceptionHandler*.cs` | Exception handling patterns |
| `logging-observability` | `**/Program.cs`, `**/*Logger*.cs` | Logging, OTel, health checks |
| `feature-flags` | `**/*Feature*.cs` | Microsoft.FeatureManagement |

### Skills por dominio

**DDD / Domain:**
- `ddd-project-planner` — Planeación DDD strategic/tactical
- `create-new-module` — Nuevo aggregate + vertical slice
- `create-value-object` — StronglyTypedId / ValueObject
- `create-specification` — Specification<T> pattern
- `implement-cqrs-command` — Write slice: command + validator + handler + endpoint
- `implement-cqrs-query` — Read slice: query + handler + endpoint + response

**ASP.NET Core:**
- `dotnet-aspnetcore/` — Web API, OpenAPI, OTel, file upload
- `dotnet-blazor/` — Blazor (10 skills: component, BFF, forms, auth, prerendering, JS interop)

**Data:**
- `dotnet-data/` — EF Core optimization, data-driven ASP.NET
- `efcore-patterns` — EF Core best practices (auto-load)

**Testing:**
- `dotnet-test/` — Code testing, run-tests, platform-detection, anti-patterns, coverage
- `dotnet-test-migration/` — MSTest/xUnit upgrades, VSTest→MTP
- `aspire-testing` — .NET Aspire integration testing

**MSBuild:**
- `dotnet-msbuild/` — Build perf, binlogs, targets, properties, items, anti-patterns (16 skills)

**AI / MCP:**
- `dotnet-ai/` — Technology selection, MCP servers (create, debug, test, publish)

**Templates:**
- `dotnet-template-engine/` — Discovery, instantiation, authoring, validation

**Diagnostics:**
- `dotnet-diag/` — Performance analysis, microbenchmarking, trace, dump collection

**Upgrade:**
- `dotnet-upgrade/` — .NET 8→9→10→11, nullable, AOT

**Cross-cutting:**
- `exception-handling` — Global exception handling (auto-load)
- `feature-flags` — Feature toggles (auto-load)
- `logging-observability` — Structured logging (auto-load)
- `minimal-ui-design-system` — Apple-inspired UI tokens
- `ngrx-signal-store` — Angular state management (auto-load)

## Estructura

```text
.agents/
├── AGENTS.md              # Regla: mantener README sincronizado
├── agents/                # Definiciones de agentes
│   ├── orchestrator.md
│   ├── planner.md
│   ├── coder.md
│   └── tester.md
└── skills/                # Skills por dominio
    ├── oro-libraries/     # MANDATORY .NET
    ├── dotnet-core/       # .NET conventions
    ├── dotnet-aspnetcore/ # ASP.NET Core
    ├── dotnet-blazor/     # Blazor (10 skills)
    ├── dotnet-data/       # EF Core
    ├── dotnet-test/       # Testing
    ├── dotnet-test-migration/ # Test migration
    ├── dotnet-msbuild/    # MSBuild (16 skills)
    ├── dotnet-ai/         # AI/MCP
    ├── dotnet-diag/       # Diagnostics
    ├── dotnet-upgrade/    # Upgrade
    ├── dotnet-template-engine/ # Templates
    ├── dotnet-maui/       # MAUI (8 skills)
    ├── dotnet-nuget/      # NuGet/CPM
    ├── dotnet/            # Scripts, P/Invoke, SDK setup, vectorization
    ├── dotnet11/          # .NET 11 specific
    ├── ddd-project-planner/
    ├── create-new-module/
    ├── create-value-object/
    ├── create-specification/
    ├── implement-cqrs-command/
    ├── implement-cqrs-query/
    ├── efcore-patterns/
    ├── exception-handling/
    ├── feature-flags/
    ├── logging-observability/
    ├── aspire-testing/
    ├── minimal-ui-design-system/
    └── ngrx-signal-store/
```

## Instalación

- **opencode**: `~/.config/opencode/agents/` + `~/.config/opencode/skills/<name>/SKILL.md`
- **Claude Code**: `~/.claude/agents/` + `~/.claude/skills/<name>/SKILL.md`
- **Codex**: `~/.codex/agents/` + `~/.codex/skills/<name>/SKILL.md`

Validar: cada skill dir contiene `SKILL.md` con `name:` que coincida con el folder; cada agent es un `.md` con `Mode:` header.
