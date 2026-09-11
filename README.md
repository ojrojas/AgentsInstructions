# AgentsInstructions

Universal **agents** + **skills** for opencode, Claude Code, Codex, Pi agent, MiniMax Code, Copilot, and any runtime supporting universal agents (markdown agent defs + native subagent delegation + file tools).

Source of truth: `.agents/agents/` (6 agents) + `.agents/skills/` (each skill owns a `SKILL.md`).

## Structure

```text
.
├── .agents/
│   ├── AGENTS.md                 # repo rule: keep this README in sync when adding/removing agents or skills
│   ├── agents/                   # subagent definitions (mode: primary | subagent)
│   │   ├── orchestrator.md       # primary — owns question tool, phases, TASKS.md gate, senior-bar enforcement
│   │   ├── planner.md            # staff architect — research-only plans, ask-ready Open Questions
│   │   ├── coder.md              # staff engineer — implements slices, LTS-baseline idioms
│   │   ├── tester.md             # senior SDET — real runs, binary Gate: PASS/BLOCKED
│   │   ├── designer.md           # senior product designer — WCAG AA, tokens, states
│   │   └── documenter.md         # senior technical writer — reports + consolidated docs/, adrs/
│   └── skills/
│       ├── oro-libraries/              # MANDATORY for .NET — vendored BuildingBlocks + CPM (has `paths`)
│       ├── dotnet-core/                # .NET backend conventions (has `paths`)
│       ├── ddd-project-planner/        # DDD strategic/tactical planning → orchestrator Tasks/Phases contract
│       ├── create-new-module/          # new aggregate + first vertical slice (canonical layout)
│       ├── implement-cqrs-command/     # write slice: command + validator + handler + endpoint
│       ├── implement-cqrs-query/       # read slice: query + handler + endpoint + response
│       ├── create-value-object/        # StronglyTypedId / ValueObject (Kernel.Domain exact types)
│       ├── create-specification/       # Specification<T> next to its aggregate
│       ├── dotnet-aspnetcore/          # Web API, OpenAPI, OTel, file upload
│       ├── dotnet-blazor/              # Blazor family (author-component, BFF, forms, prerendering…)
│       ├── dotnet-data/                # EF Core patterns + query optimization
│       ├── dotnet-ai/                  # AI/ML tech selection + C# MCP servers
│       ├── dotnet-maui/                # MAUI lifecycle, navigation, bindings, theming
│       ├── dotnet-msbuild/             # build perf, binlogs, targets, properties, items
│       ├── dotnet-nuget/               # CPM conversion, trusted publishing
│       ├── dotnet-template-engine/     # template discovery/instantiation/authoring
│       ├── dotnet-test/                # run-tests, platform-detection, anti-patterns, gaps, coverage
│       ├── dotnet-test-migration/      # MSTest/xUnit/NUnit upgrades, VSTest→MTP
│       ├── dotnet-upgrade/             # .NET 8→9→10→11, nullable, AOT, Thread.Abort
│       ├── dotnet-diag/                # traces, dumps, perf, crash symbolication
│       ├── dotnet/ dotnet11/ dotnet-advanced/ dotnet-aspnet/ dotnet-experimental/
│       ├── efcore-patterns/ exception-handling/ feature-flags/ logging-observability/
│       ├── ngrx-signal-store/          # Angular SignalStore (has `paths`)
│       ├── aspire-testing/ minimal-ui-design-system/
│       └── angular-developer@ angular-new-app@  # symlinks (currently broken self-loops — see note)
└── README.md
```

`@` = symlink. `angular-developer` / `angular-new-app` currently resolve to a self-loop
(`../../.agents/skills/...`) and are unreadable — fix or remove before relying on Angular auto-load.

## Operating contracts (what makes this repo different)

1. **Canonical .NET architecture (mandatory, fixed).** Single service = single `.csproj`;
   DDD tactical + Vertical Slices. Reference: `$HOME/Sources/BuildingBlocks/examples/Identity`
   (`Identity.Server` + `Identity.Server.Client`):
   `src/Services/{Service}/Domain/{Aggregate}/` (pure) +
   `Application/Features/{Context}/{Feature}.cs` (one file: command/query + validator +
   handler + `IEndpoint`) + `Infrastructure/` (the ONLY home of `DbContext : AppDbContextBase`,
   configurations, migrations, `EfRepository`, outbox). Forbidden: `src/Core|Application|
   Infrastructure|Server` splits, `Modules/{X}/application,domain,persistence` siblings,
   layer folders scattering a feature, EF/bus types inside `Domain/`, direct `IEventBus`
   publishes from handlers (use `IOutboxWriter.StageAsync` + single `SaveChangesAsync`).
   Frontend tracks: Angular `apps/web/src/app/{core,shared,features/{feature},shell}` or
   Blazor `{Service}.Client/` — never root type-only folders, never business logic in UI files.
2. **Language LTS baseline (coder, polyglot).** The coder resolves the effective LTS per
   language from repo pins (`global.json`/`TargetFramework`, `package.json`+`tsconfig`,
   `pom.xml`/`build.gradle`, …) vs. installed SDKs vs. official support calendars, codes
   with that LTS's idioms only, declares `Toolchain | Baseline | Features | Source` in
   `NOTES.md`, and never uses preview/STS features or silent major upgrades.
3. **Question-tool ownership.** ONLY the orchestrator may invoke the harness question tool
   (opencode `question`, Claude Code `AskUserQuestion`, harness equivalent) — Step 0 gate
   before planning + re-ask loop for avoidable planner unknowns. Plain-text questions are
   forbidden. Subagents never address the user; they emit **ask-ready** items
   (`Qxx [BLOCKING|OPTIONAL — default: X] — question | Options: A) recommended…`)
   in `Open Questions` / reports and proceed on defaults.
4. **Senior bar (blocking).** Each agent is its craft's most senior practitioner
   (planner = staff architect, coder = staff engineer, tester = senior SDET,
   designer = senior product designer, documenter = senior technical writer).
   The orchestrator rejects below-bar output (`BLOCKED (<agent> below senior bar)`)
   instead of patching it; every agent self-checks the bar before returning.
5. **Machine-readable execution.** Planner output always ends in `Tasks` table + `Phases` +
   `TASKS.md` checklist with identical schema in legacy and SDD modes, so the orchestrator
   can phase, parallelize (`max_parallel: 3`, no overlapping WRITEs per phase), and gate
   task-by-task on evidence files.

## Installation (per provider)

No `setup.sh` in this repo. Install by symlink (dev) or copy (distribution):

- **opencode**: `~/.config/opencode/agents/` + `~/.config/opencode/skills/<name>/SKILL.md`
- **Claude Code**: `~/.claude/agents/` + `~/.claude/skills/<name>/SKILL.md`
- **Codex**: `~/.codex/agents/` + `~/.codex/skills/<name>/SKILL.md`
- **Copilot**: project `.github/skills/<name>/SKILL.md` (or global `~/.copilot/skills/`)
- **Pi / MiniMax**: reference repo-local `.agents/agents/` + `.agents/skills/` directly

Validate after install: every skill dir contains `SKILL.md` whose `name:` matches the folder;
every agent is a single `.md` with a `Mode:` header.

## Usage

### Agents

Via the runtime's native subagent mechanism (Claude Code `/agent <name>`, opencode `task`,
Codex/Pi/MiniMax subagent call). Flow: **Orchestrator** (Step 0 questions →) **Planner**
(`PLAN.md` + `TASKS.md`) → phases of **Coder/Designer** → **Tester** (`Gate: PASS`) →
**Documenter** → consolidated `docs/` + `adrs/`.

| Agent | Mode | Role |
|---|---|---|
| `orchestrator` | primary | coordinates, owns question tool + `TASKS.md`, enforces senior bar |
| `planner` | subagent | research-only plans (WHAT, never HOW bodies); SDD-aware |
| `coder` | subagent | implements tasks, honors `PLAN.md` paths + LTS baseline |
| `tester` | subagent | runs tests, writes `TEST-REPORT.md` with binary gate |
| `designer` | subagent | component contracts, WCAG AA gate |
| `documenter` | subagent | per-task `DOC-REPORT.md` + final `docs/` consolidation |

### Skills

- **Auto-load (`paths` frontmatter):** `**/*.cs*` → `oro-libraries`, `dotnet-core`,
  `efcore-patterns`; `**/*.ts` → `ngrx-signal-store`; Blazor/Test family per skill paths.
  Fallback: repo-local `.agents/skills/`.
- **On demand:** `/<skill-name>` (e.g. `/implement-cqrs-command`, `/ddd-project-planner`).
- **Planning trigger:** business-idea / SaaS / ERP / bounded-contexts / ADRs / C4 language,
  or explicit SDD tokens (`sdd`, `spec-kit`, `openspec`, `constitution`, `/sdd`), loads
  `ddd-project-planner` via the planner (SDD toolchain detected read-only in Phase 0.6).

### Artifacts (`draft/`)

Created by the orchestrator in the **consuming repo** (not here):

```text
draft/{YYYYMMDD}/
├── plans/00-{plan-slug}/   # PLAN.md (immutable), PLAN-v2.md, TASKS.md (blocking), docs/, adrs/
├── tasks/{NN}-{slug}/      # NOTES.md, TEST-REPORT.md, DOC-REPORT.md + code evidence
└── notes/                  # cross-cutting notes
```

`TASKS.md` gates strictly: task N+1 never starts until task N's block is fully `[x]`
on disk evidence. Only the orchestrator mutates it.

## Contributing

Per `.agents/AGENTS.md`: when adding/removing skills or agents, update this README's
Structure/Skills tables so downstream repos can copy/paste them.
