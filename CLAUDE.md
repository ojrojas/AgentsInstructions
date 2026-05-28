# AgentsInstructions

This project provides **agents** and **skills** for Claude Code following the standard Claude Code Skills structure.

Rules have been merged into skills using the `paths` frontmatter field for auto-loading.

## Agents

Specialized agents in `.claude/agents/`:

| Agent | File | Purpose |
|---|---|---|
| Coder | `.claude/agents/coder.md` | Writes code following mandatory coding principles |
| Designer | `.claude/agents/designer.md` | Handles all UI/UX design tasks |
| Orchestrator | `.claude/agents/orchestrator.md` | Coordinates complex feature implementations |
| Planner | `.claude/agents/planner.md` | Creates implementation plans |
| Backend C# | `.claude/agents/backend-csharp.md` | .NET DDD/CQRS backend development |
| Frontend Angular | `.claude/agents/frontend-angular.md` | Angular/ngRx Signals frontend development |
| Fullstack Identity | `.claude/agents/fullstack-identity.md` | End-to-end identity server development |
| Testing Engineer | `.claude/agents/testing-engineer.md` | Automated testing across the stack |

## Skills

Reusable skills in `.claude/skills/`:

| Skill | Description | Auto-loads On |
|---|---|---|
| `angular` | Angular development best practices | `**/*.ts`, `src/app/**/*.html` |
| `angular-create-feature` | Scaffold Angular feature modules | — |
| `architecture` | Clean Architecture guidance | `**/*.cs`, `**/*.csproj`, `**/*.slnx` |
| `oro-libraries` | OroCQRS, OroBuildingBlocks, OroKernel integration + CPM | `**/*.cs`, `**/*.csproj`, `**/*.slnx`, `**/*.props` |
| `backend-netcore` | .NET DDD/CQRS backend scaffolding | `**/*.cs`, `**/*.csproj` |
| `blazor` | Blazor client-server patterns | `**/*.razor`, `**/*.cs` |
| `conventions` | Coding conventions reference | `**/*.cs`, `**/*.ts`, `**/*.razor`, `**/*.csproj` |
| `create-endpoint` | Minimal API endpoint templates | — |
| `create-new-module` | Complete DDD module scaffolding | — |
| `create-specification` | Specification pattern for queries | — |
| `create-value-object` | Strongly-typed Value Objects | — |
| `fluentui-blazor` | Fluent UI Blazor component library | `**/*.razor` |
| `implement-cqrs-command` | CQRS command implementation | — |
| `implement-cqrs-query` | CQRS query implementation | — |
| `ngrx-signal-store` | NgRx SignalStore state management | `**/*.ts`, `**/*.store.ts` |
| `project-rules` | Strict development and process rules | `**/*.cs`, `**/*.ts`, `**/*.razor`, `**/*.csproj` |
| `testing` | Testing strategy and patterns | `**/*.cs`, `**/*.ts`, `**/*.csproj` |

## Usage

1. Install globally: `./setup.sh`
2. Skills auto-load based on project file types (when `paths` is set)
3. Load an agent with `/agent <name>`
4. Invoke a skill directly with `/<skill-name>`
5. The orchestrator agent delegates to specialist agents
