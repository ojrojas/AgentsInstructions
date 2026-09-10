# AgentsInstructions

This project provides **universal agents** and **skills** (markdown agent defs + SKILL.md) for opencode, Claude Code, Codex, Pi agent, MiniMax Code, Copilot, and any runtime supporting universal agents.

Rules have been merged into skills using the `paths` frontmatter field for auto-loading.

## Agents

Specialized agents in `.claude/agents/`:

| Agent | File | Purpose |
|---|---|---|
| Coder | `.claude/agents/coder.md` | Writes code following mandatory coding principles |
| Designer | `.claude/agents/designer.md` | Handles all UI/UX design tasks |
| Orchestrator | `.claude/agents/orchestrator.md` | Coordinates complex feature implementations |
| Planner | `.claude/agents/planner.md` | Creates implementation plans |
| Documenter | `.claude/agents/documenter.md` | Architecture docs, ADRs, API references |
| Tester | `.claude/agents/tester.md` | Automated testing across the stack |

## Skills

Reusable skills in `.claude/skills/`:

| Skill | Description | Auto-loads On |
|---|---|---|
| `angular-developer` | Angular best practices, scaffolding, signals, routing, testing | `**/*.ts`, `src/app/**/*.html` |
| `oro-libraries` | Vendored BuildingBlocks (CQRS, Kernel.Domain/Infrastructure, EventBus/RabbitMQ, ServiceDefaults, Logger) + CPM for externals | `**/*.cs`, `**/*.csproj`, `**/*.slnx`, `**/*.props` |
| `author-component` | Blazor component architecture | `**/*.razor`, `**/*.cs` |
| `create-new-module` | Complete DDD module scaffolding | — |
| `create-specification` | Specification pattern for queries | — |
| `create-value-object` | Strongly-typed Value Objects | — |
| `implement-cqrs-command` | CQRS command implementation | — |
| `implement-cqrs-query` | CQRS query implementation | — |
| `ngrx-signal-store` | NgRx SignalStore state management | `**/*.ts`, `**/*.store.ts` |

## Usage

1. Install per provider (opencode / Claude Code / Codex / Pi / MiniMax — see `README.md`; no single `setup.sh` is assumed)
2. Skills auto-load based on project file types (when `paths` is set) or via your runtime's native skill mechanism, fallback to repo-local `.claude/skills/`
3. Load an agent via your runtime's native subagent mechanism
4. Invoke a skill directly with `/<skill-name>` (or runtime equivalent)
5. The orchestrator agent delegates to specialist agents
