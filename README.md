# AgentsInstructions

This project provides **agents** and **skills** for **Claude Code**, following the standard Claude Code Skills structure.

Rules have been merged into skills using the `paths` frontmatter field for auto-loading.

## Structure

```
.
├── CLAUDE.md                  # Root instructions
├── .claude/
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
│       ├── oro-libraries/             # OroCQRS, OroBuildingBlocks, OroKernel + CPM
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

## Installation

```bash
./setup.sh
```

Running `setup.sh` creates symlinks in `~/.claude/` so agents and skills are available globally.

## Usage

### Skills (auto-load with paths)

Skills with a `paths` frontmatter field auto-load when Claude Code detects matching file types:

- `.cs`, `.csproj`, `.slnx`, `.props` files → architecture, backend-netcore, conventions, project-rules, testing, oro-libraries
- `.ts`, `.html` files → angular, ngrx-signal-store, conventions, project-rules, testing
- `.razor` → blazor, fluentui-blazor, conventions, project-rules
- Test files → testing

Skills without `paths` are invoked on demand via `/<skill-name>`.

### Agents

Load an agent in Claude Code with:

```
/agent <name>
```

Available agents: `coder`, `designer`, `documenter`, `orchestrator`, `planner`, `tester`.

### Skills

Invoke a skill directly with:

```
/<skill-name>
```

## External Plugins

- **dotnet-skills** — 167+ .NET development skills and 16 specialized agents by Aaron Stannard. See `dotnet-skills/README.md`.
