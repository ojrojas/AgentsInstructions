# AgentsInstructions

This project provides **universal agents** and **skills** for opencode, Claude Code, Codex, Pi agent, MiniMax Code, Copilot, and any runtime supporting universal agents.

Rules have been merged into skills using the `paths` frontmatter field for auto-loading.

## Structure

```
.
├── CLAUDE.md                  # Root instructions
├── .agents/
│   ├── agents/                # Agent definitions
│   │   ├── coder.md
│   │   ├── designer.md
│   │   ├── documenter.md
│   │   ├── orchestrator.md
│   │   ├── planner.md
│   │   └── tester.md
│   └── skills/                # Skills (each directory or sub-directory has SKILL.md)
│       ├── create-new-module/          # DDD module scaffolding
│       ├── create-specification/       # Specification pattern
│       ├── create-value-object/        # Value Objects
│       ├── oro-libraries/             # Vendored BuildingBlocks (CQRS, Kernel, EventBus, ServiceDefaults, Logger) + CPM
│       ├── dotnet-ai/                 # AI/ML & MCP servers
│       ├── dotnet-aspnet/             # ASP.NET Core Web APIs
│       ├── dotnet-blazor/             # Blazor components & patterns
│       ├── dotnet-data/               # EF Core optimization
│       ├── dotnet-diag/               # Diagnostics, performance, crash analysis
│       ├── dotnet-experimental/       # Experimental: SIMD, mock analysis
│       ├── dotnet-maui/               # .NET MAUI mobile/desktop
│       ├── dotnet-msbuild/            # MSBuild, build perf, binlogs
│       ├── dotnet-nuget/              # NuGet CPM, publishing
│       ├── dotnet-template-engine/    # dotnet new templates
│       ├── dotnet-test/               # Testing, coverage, mocks
│       ├── dotnet-upgrade/            # Migration between .NET versions
│       ├── dotnet/                    # General .NET (P/Invoke, scripts)
│       ├── dotnet11/                  # .NET 11 specific APIs
│       ├── efcore-patterns/           # EF Core best practices
│       ├── exception-handling/        # ASP.NET Core error handling
│       ├── feature-flags/             # Feature management
│       ├── implement-cqrs-command/    # CQRS command scaffold
│       ├── implement-cqrs-query/      # CQRS query scaffold
│       ├── logging-observability/     # Serilog, OpenTelemetry
│       └── ngrx-signal-store/         # Angular state management
├── dotnet-skills/             # External .NET skills plugin (submodule)
├── setup.sh                   # Installation script
└── README.md                  # This file
```

## Installation (per provider)

No single `setup.sh` is assumed (legacy reference removed — the script does not exist in this repo).
Install per runtime, pointing at repo-local `.claude/agents/` + `.claude/skills/`:

- **opencode**: symlink or copy to `~/.config/opencode/agents/` + `~/.config/opencode/skills/`, or reference repo-local path.
- **Claude Code**: symlink to `~/.claude/agents/` + `~/.claude/skills/`.
- **Codex**: symlink to `~/.codex/agents/` + `~/.codex/skills/` (or repo-local reference per your setup).
- **Pi agent / MiniMax Code**: reference repo-local `.claude/agents/` + `.claude/skills/` directly (no global install assumed).

## Usage

### Skills (auto-load with paths)

Skills with a `paths` frontmatter field auto-load when your runtime detects matching file types (Claude Code / opencode paths-aware loaders; otherwise load manually with fallback to repo-local `.claude/skills/`):

- `.cs`, `.csproj`, `.slnx`, `.props` files → oro-libraries (vendored BuildingBlocks), dotnet-core, efcore-patterns
- `.ts`, `.html` files → angular-developer, ngrx-signal-store
- `.razor` → author-component and the dotnet-blazor skill family
- Test files → code-testing-agent, run-tests

Skills without `paths` are invoked on demand via `/<skill-name>`.

### Agents

Load an agent via your runtime's native subagent mechanism (e.g. Claude Code `/agent <name>`, opencode `task`, Codex/Pi/MiniMax subagent call):

Available agents: `coder`, `designer`, `documenter`, `orchestrator`, `planner`, `tester`.

### Artifacts (`draft/`)

The orchestrator persists all execution artifacts under a dated, type-separated tree:

```
draft/{YYYYMMDD}/
├── plans/00-{plan-slug}/   # PLAN.md, TASKS.md (blocking checklist), docs/, adrs/
├── tasks/{NN}-{slug}/      # NOTES.md, TEST-REPORT.md, DOC-REPORT.md
└── notes/                  # cross-cutting notes
```

`TASKS.md` gates execution task-by-task: a task's checks must all be marked before the
next task starts. See `.claude/agents/orchestrator.md`.

### Skills

Invoke a skill directly with:

```
/<skill-name>
```

## External Plugins

- **dotnet-skills** — 167+ .NET development skills and 16 specialized agents by Aaron Stannard. See `dotnet-skills/README.md`.
