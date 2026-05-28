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
│   │   ├── orchestrator.md
│   │   ├── planner.md
│   │   ├── backend-csharp.md
│   │   ├── frontend-angular.md
│   │   ├── fullstack-identity.md
│   │   └── testing-engineer.md
│   └── skills/                # Skills (each directory has SKILL.md)
│       ├── angular/
│       ├── angular-create-feature/
│       ├── architecture/
│       ├── backend-netcore/
│       ├── blazor/
│       ├── conventions/
│       ├── create-endpoint/
│       ├── create-new-module/
│       ├── create-specification/
│       ├── create-value-object/
│       ├── fluentui-blazor/
│       ├── implement-cqrs-command/
│       ├── implement-cqrs-query/
│       ├── ngrx-signal-store/
│       ├── project-rules/
│       └── testing/
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

- `.cs`, `.csproj`, `.slnx` files → architecture, backend-netcore, conventions, project-rules, testing
- `.ts`, `.html` files → angular, ngrx-signal-store, conventions, project-rules, testing
- `.razor` → blazor, fluentui-blazor, conventions, project-rules
- Test files → testing

Skills without `paths` are invoked on demand via `/<skill-name>`.

### Agents

Load an agent in Claude Code with:

```
/agent <name>
```

Available agents: `coder`, `designer`, `orchestrator`, `planner`, `backend-csharp`, `frontend-angular`, `fullstack-identity`, `testing-engineer`.

### Skills

Invoke a skill directly with:

```
/<skill-name>
```

## External Plugins

- **dotnet-skills** — 167+ .NET development skills and 16 specialized agents by Aaron Stannard. See `dotnet-skills/README.md`.
