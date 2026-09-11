---
name: ddd-project-planner
description: Turns a business idea into a complete, ready-to-build project plan using Domain-Driven Design — domain discovery, bounded contexts, context map, ubiquitous language, aggregates/entities/value objects, architecture recommendation, backlog, user stories with Given/When/Then acceptance criteria, TDD strategy, and a sprint roadmap. Use this skill whenever the user wants to plan a new software product, describes a business idea and asks for a technical plan, asks for DDD modeling (bounded contexts, aggregates, ubiquitous language, context map), asks to turn an idea into a backlog/roadmap/sprints, or mentions planning a SaaS, ERP, CRM, marketplace, or enterprise system from scratch — even if they don't use the words "DDD" or "planner" explicitly. Also use it if the user asks to add "enterprise-level" rigor to a plan (ADRs, Event Storming, C4 model, NFR matrix, traceability matrix). Emits stable US-xx/FR-xx/NFR-xx IDs; when .NET is detected forces DDD + Vertical Slices (Features/{Context}/{Feature}/) per repo contract; appends an orchestrator-compatible Tasks/Phases + TASKS.md checklist appendix. Enterprise additions are a subset of SDD mode, not a separate trigger. Load explicitly via SDD request or Planner delegation (no `paths` auto-load).
---

# DDD Project Planner

Turns a business idea into a complete project plan: domain model, architecture, backlog, user
stories, TDD strategy, and sprint roadmap — delivered as a single Markdown document.

**SDD toolchain precedence**: when the Planner reports Phase 0.6 = `speckit` or `openspec`, this skill supplies the DOMAIN CONTENT (ubiquitous language, contexts, model, US/FR/NFR, ADRs) mapped into the tool's native artifacts (`specs/*/spec.md + plan.md + tasks.md` for spec-kit; `openspec/changes/*/proposal.md + specs + tasks.md` for OpenSpec) — it does NOT replace the tool's flow. Only when Phase 0.6 = `none (manual)` does this skill's own document structure act as the spec source of truth (plus the mandatory Appendix A execution contract).

## Step 1 — Gather input

Before planning, make sure you have (ask only for what's missing — don't re-ask what the user
already told you, and don't block on nice-to-haves):

- **Business description**: what does the product do, who is it for
- **Product type**: SaaS, Enterprise, Marketplace, ERP, CRM, internal tool, etc.
- **Users**: main user roles/personas
- **Technical constraints**: anything that's fixed (compliance, existing systems to integrate, deadlines)
- **Tech stack**: preferred or required languages/frameworks (if none given, propose one in Step 4 and say so explicitly)
- **Expected scale**: rough user/traffic volume, or "not sure yet" is fine

If the user gives you a rich description up front, extract these from it rather than asking again.
Only ask a clarifying question when a genuinely different plan would result depending on the
answer (e.g., B2B vs B2C changes the whole domain model). Otherwise state your assumption inline
and proceed.

**Enterprise mode**: turn on the "Enterprise" additions (see Step 7) only if the user explicitly
signals this is a corporate/enterprise-scale project — e.g. they used the word "enterprise",
mention multiple integrated systems, compliance/audit needs, a large org, or ask for ADRs/Event
Storming/C4/NFRs by name. Otherwise, skip Step 7 entirely and keep the plan lean.

## Step 2 — Business analysis

Work through this like a Business Analyst would, and capture it briefly at the top of the output:

- **Objectives**: what business outcome this product exists to achieve
- **Stakeholders**: who cares about this system and why
- **Key processes**: the main business processes the system supports
- **Business rules**: constraints/policies that shape the domain (these often become domain invariants later)
- **Risks**: technical, business, or adoption risks worth flagging early

## Step 3 — Domain discovery and DDD strategic design

Think through this like a Domain Expert followed by a DDD Architect:

1. **Domain classification** — identify the Core Domain (the competitive-advantage part),
   Supporting Domains, and Generic Domains (solved problems, e.g. auth, email — candidates for
   off-the-shelf solutions).
2. **Ubiquitous Language** — a short glossary of domain terms as the business would say them, used
   consistently for the rest of the document.
3. **Bounded Contexts** — split the domain into contexts (e.g. Identity, Billing, Inventory,
   Sales, Notifications, Reporting). Name each one, state its responsibility, and note which
   domain classification it belongs to (core/supporting/generic).
4. **Context Map** — show how contexts relate and depend on each other (e.g. Customer → Sales →
   Billing → Accounting), including the relationship type where relevant (Shared Kernel,
   Customer-Supplier, Anti-Corruption Layer, etc.) if the user's domain calls for that level of detail.

## Step 4 — Domain modeling and architecture

For each bounded context that matters to an MVP (don't force-model trivial/generic contexts in
detail), produce:

- **Entities** — objects with identity and lifecycle
- **Value Objects** — immutable objects defined by their attributes
- **Aggregates** — consistency boundaries, with their aggregate root
- **Domain Events** — meaningful things that happen (e.g. `OrderPlaced`, `InvoiceIssued`)
- **Domain Services / Policies** — logic that doesn't naturally belong to one entity

Then propose the application and infrastructure shape (canon: `examples/Identity/Identity.Server`):

- **Application layer**: commands, queries, handlers — keep this to a representative sample per
  context, not exhaustive. .NET: slices live at `src/Services/{Service}/Application/Features/{Context}/{Feature}.cs`
  (command/query + validator + handler + `IEndpoint` in ONE file). Never `src/Application/Modules/.../Commands|Queries|DTOs`.
- **Infrastructure needs**: persistence, messaging, cache, external APIs, storage, email — only
  what the domain actually requires. .NET: EVERYTHING persistence-related lives under
  `src/Services/{Service}/Infrastructure/` (`Persistence/{Service}DbContext.cs : AppDbContextBase`,
  `Persistence/Configurations/`, `Persistence/Migrations/`, repository impls `: EfRepository<T,TId>`,
  `External/` adapters). A sibling `Persistence/` folder next to `Domain/`/`Application/`, or any EF
  type inside `Domain/`, is FORBIDDEN (this was the reported `modules/catalog/{x}` bug).
- **Architecture style recommendation**:
  - Non-.NET stacks: pick one (Clean Architecture, Hexagonal/Ports & Adapters, Modular Monolith, Microservices, Vertical Slice) based on project size and team size, and briefly justify — don't default to microservices for a small project.
  - **.NET override (mandatory, repo contract)**: architecture is ALWAYS **single-service-project DDD tactical + Vertical Slices**. No Clean/Hexagonal/Microservices proposal for .NET, no `src/Core + src/Application + src/Infrastructure + src/Server` multi-csproj split, no `Modules/{X}/application,domain,persistence` siblings. Tactical mapping: `AggregateRoot<Entity<TId>>` + `StronglyTypedId`, `CheckRule`/`RaiseDomainEvent`, `Result`/`Error` returns, `Specification<T>` queries (`Where(...)`), `AppDbContextBase` + `AddUnitOfWork` + `AddOutbox` (`StageAsync` then single `SaveChangesAsync`), dispatch via `ISender.SendAsync`, HTTP via `Result → HTTP` extensions. Externals via CPM.
  - **Frontend track (declare one when UI exists)**: Angular SPA → `apps/web/src/app/{core/,shared/,features/{feature}/,shell/}` (component+service+store+routes+spec together per feature, `ngrx-signal-store`); Blazor → `{Service}.Client/{Pages/,Services/*ApiClient.cs,Models/Contracts.cs}` + server shell `Components/`. Never type-only root folders; never business logic or EF in `.razor`/`.component.ts` — UI calls slice HTTP APIs.
- **Tech stack**: use what the user specified; if unspecified, propose a stack and say it's a suggestion the user can swap out. When .NET is specified or detected, assume `oro-libraries` BuildingBlocks context.

## Step 4b — Slice adapter (mandatory for .NET, recommended otherwise)

Translate each MVP bounded context into executable slices BEFORE writing backlog IDs.
Canon: `examples/Identity/Identity.Server/Application/Features/Users/RegisterUser.cs`.

- One feature = one file `src/Services/{Service}/Application/Features/{Context}/{Feature}.cs` with command/query + validator + handler + `IEndpoint` (+ response DTO where needed). Split into a `Features/{Context}/{Feature}/` folder ONLY when the slice outgrows one file. Never plan layer folders (`Commands/`, `Handlers/`, `Queries/`, `Repositories/`, `Controllers/`, `Endpoints/`, `Validators/`) and never `src/Core/Modules + src/Application/Modules + src/Infrastructure + src/Server/EndPoints` (retired `create-new-module` layout) nor `Modules/{X}/application,domain,persistence` siblings.
- Domain purity: `src/Services/{Service}/Domain/{Aggregate}/` holds ONLY aggregate + `StronglyTypedId` + `ValueObject` + `Enumeration` + `IBusinessRule` + domain events + `Specification<T>` + repository/domain-service INTERFACES. Persistence (`DbContext : AppDbContextBase`, `IEntityTypeConfiguration<>`, migrations, `EfRepository` impls, `OutboxEntityTypeConfiguration`) lives ONLY under `src/Services/{Service}/Infrastructure/` (prefer `Infrastructure/Persistence/`).
- Name every story so its slice file is derivable (`{Context}/{Feature}` → `{Feature}Command`, `{Feature}Validator`, `{Feature}Handler`, `{Feature}Endpoint` in `{Feature}.cs`).
- Record one-liner signatures the Coder needs (no bodies, max one line each), e.g. `ISender.SendAsync<T>(IRequest<T>, ct)`, `IEndpoint.MapEndpoint(IEndpointRouteBuilder)`, `IOutboxWriter.StageAsync(IntegrationEvent, ct)`, `IUnitOfWork.SaveChangesAsync(ct)`.
- Non-.NET: same rule adapted — one feature = one folder (`features/{feature}/` with components/services/routes/tests together).

## Step 5 — Backlog and user stories

Structure: **Epic → Features → Stories → Tasks**, with stable IDs throughout (`US-01…`, `FR-01…`, `NFR-01…`). Every `FR/NFR` MUST map to at least one task ID later (or be listed as uncovered → Open Question).

For each user story use this exact template (ID required):

```
### [Story ID] Story title

**As** <role>
**I want** <capability>
**So that** <business value>

**Acceptance Criteria**
- Given <context>, When <action>, Then <outcome>
- (add more Given/When/Then lines as needed)
```

Group stories by epic, and make sure every epic maps back to a bounded context from Step 3 so the
backlog and the domain model stay traceable to each other.

## Step 6 — TDD strategy and sprint roadmap

**TDD plan (Tester-consumable)**: for each major use case, list tests across levels — Unit, Integration,
Contract, E2E — as `Test notes` per future task (e.g. `unit: validator + handler Result paths; integration: POST round-trip + outbox StageAsync → processor → bus`). Note Red → Green → Refactor applies. .NET defaults: xUnit + Moq + coverlet via CPM, `Specification.IsSatisfiedBy` in unit tests, SQLite/Testcontainers over InMemory for EF integration. Don't write test code; this is a test *plan*.

**Sprint roadmap + Phase mapping**: group epics into sprints in dependency order (e.g.
Authentication & Users first). For each sprint list: goal, contexts/features covered, dependencies. Additionally record a `Phases` derivation rule the Planner will use verbatim: `PARALLEL` when no overlapping written `Files` AND no data dependency; `SEQUENTIAL` when task B needs A's output, both write the same file, or shared files (`Program.cs`, `DbContext`, `Directory.Packages.props`) are touched. Sprints inform priority; Phases govern execution.

## Step 7 — Enterprise additions (only if Enterprise mode is on)

Add these sections when triggered (see Step 1):

- **Event Storming summary** — key domain events in business-process order (Big Picture level),
  plus a Design-Level pass for the core domain's most complex flow
- **ADRs (Architecture Decision Records)** — one short ADR per major architectural decision made in
  Step 4 (context, decision, consequences)
- **Quality Attributes / NFR matrix** — a table of non-functional requirements (performance,
  availability, security, maintainability, observability, scalability) with a target and how it's
  addressed
- **Traceability matrix** — a table linking business objectives → epics → stories → use cases, so
  every story can be traced back to a business reason
- **C4 model summary** — Context and Container level descriptions (Component/Code level only if the
  user asks, since it needs actual code to be meaningful)
- **DevOps plan** — CI/CD approach, containerization, infra-as-code, observability (logging,
  tracing), feature flags
- **Security plan** — auth approach (OAuth2/OIDC/JWT), authorization model (RBAC/ABAC), audit
  logging, encryption at rest/in transit, secrets management, and an OWASP Top 10 checklist relevant
  to this system

## Output

Deliver the narrative plan as **one Markdown document** (use the docx skill instead only if the user
explicitly asks for Word), with clear `##` headers in this order:

```
1. Vision & Business Analysis
2. Ubiquitous Language
3. Domain Discovery (Core/Supporting/Generic)
4. Bounded Contexts & Context Map
5. Domain Model (per context: entities, value objects, aggregates, events)
6. Architecture (style + justification, application layer, infrastructure; .NET → fixed Vertical Slice + DDD per Step 4 override)
7. Backlog (epics → features → stories with US-xx IDs + Given/When/Then acceptance)
8. Requirements traceability (FR-01…/NFR-01… each mapped to ≥1 future task ID or marked uncovered → Open Question)
9. TDD Strategy (as Tester-consumable Test notes)
10. Sprint Roadmap (+ Phase derivation rule)
11. [Enterprise/SDD only] Event Storming, ADRs, NFR Matrix, Traceability Matrix, C4 Summary, DevOps Plan, Security Plan
12. Risks & Technical Debt
```

Then append **Appendix A — Orchestrator contract (mandatory, machine-readable)** so the Planner can paste it verbatim into `Tasks` + `Phases` + `TASKS.md` without re-deriving:

```markdown
## Tasks

| ID | Description (WHAT outcome) | Files (created / modified, exact repo-relative) | Agent | Depends_on | Draft | Acceptance criteria | Test notes | Signatures (optional, one-liners, no bodies) |
|---|---|---|---|---|---|---|---|---|
| T01 | ... | creates `src/Services/{Service}/Application/Features/{Context}/{Feature}.cs` (+ `Domain/{Aggregate}/` files where new aggregate) | Coder | — | `draft/{YYYYMMDD}/tasks/01-{slug}` | observable pass/fail | Tester-consumable notes | `ISender.SendAsync<T>(IRequest<T>, ct)` |

## Phases

### Phase 1: [Name] (PARALLEL — no overlapping WRITES, no data dependency)
- T01 → Coder (Files: ...)

### Phase 2: [Name] (depends on Phase 1, SEQUENTIAL)
- T02 → Tester (Files: ...)

# TASKS — {plan-slug}

## [ ] T01 — {outcome}
- [ ] Implementation in `tasks/01-{slug}/` (Files: `...`)
- [ ] Acceptance criteria: {...}
- [ ] `TEST-REPORT.md` → `Gate: PASS`
- [ ] `DOC-REPORT.md` → `Gate: PASS`
- [ ] `NOTES.md` recorded
```

Sizing rules for the appendix: one task = one concern, 1–4 files, independently testable; never overlap WRITES in one phase; shared files (`Program.cs`, `DbContext`, `Directory.Packages.props`, root configs) force sequential tasks; test work is its own task.

Keep every section proportional to size. Trigger note: `enterprise` wording alone does not switch modes — Enterprise additions (§11) render only when the SDD/enterprise bar is met (multi-system, compliance/audit, or explicit ADRs/C4/NFR request); otherwise skip and keep the plan lean.

## Self-check (run before returning)

- [ ] Every `US-xx` has `Given/When/Then`; every `FR/NFR` traces to ≥1 task ID or is marked uncovered.
- [ ] .NET plans use single-file slices `src/Services/{Service}/Application/Features/{Context}/{Feature}.cs` only (no layer folders, no `src/Core|Application|Infrastructure|Server` split, no `Modules/{X}/application,domain,persistence` siblings; persistence only under `Infrastructure/`).
- [ ] Appendix A present with exact `Files`, valid `Draft` paths, acceptance + test notes, one-line signatures only.
- [ ] No phase overlaps WRITES; dependencies acyclic.
