# dotnet-mvc-skills

[繁體中文](./README.zh-TW.md)

A [Claude Code plugin](https://docs.anthropic.com/en/docs/claude-code/plugins) for opinionated C# .NET 10 MVC development — four-tier architecture (Web / Service / Data / Library), UnitOfWork + IDbContextFactory, TDD workflows, SOLID principles, and structured code review.

## Install

```bash
# add as marketplace + install (recommended)
/plugin marketplace add {your-github-username}/dotnet-mvc-skills
/plugin install dotnet-mvc-skills

# or load directly during development
claude --plugin-dir ./dotnet-mvc-skills
```

Skills: `/write`, `/fix`, `/review`, `/refactor`, `/debug`, `/spec`, `/decision`

## Skills

### Workflow Skills (manually invoked)

| Skill | Usage |
|-------|-------|
| `/write <feature>` | Implement a feature with TDD |
| `/fix <bug>` | Fix a bug (diagnose → Red → Green → Refactor) |
| `/review [--staged \| path]` | Review local changes (style, tests, architecture) |
| `/refactor [path \| module]` | Safe refactoring with smell analysis and TDD verification |
| `/debug <error>` | Systematic root cause investigation before any fix |
| `/spec <feature>` | Define interface contract (Given/When/Then + C# interface + invariants) |
| `/decision <A vs B>` | AI-era tech decision framework (4-dimension scoring + pre-mortem + exit plan) |

### Methodology Skills (auto-loaded by workflows)

These are not shown in the `/` menu. Claude loads them automatically when needed.

| Skill | When auto-loaded |
|-------|-----------------|
| `principles` | Designing features, architecture decisions, SOLID violations |
| `testing` | Implementing features, fixing bugs, changing behavior |
| `done` | End of any workflow that produces code changes |

### Domain Skills (auto-loaded by context)

Specialized .NET MVC knowledge loaded automatically when relevant.

| Skill | Coverage |
|-------|----------|
| `three-tier-architecture` | Four-tier layer responsibilities, dependency direction, file structure |
| `unitofwork-dbcontextfactory` | UnitOfWork pattern, IDbContextFactory, parallel queries, background services |
| `efcore-patterns` | Entity configuration, migrations, query optimization, Global Query Filters |
| `database-performance` | Read/write separation, N+1 prevention, AsNoTracking, row limits |
| `csharp-coding-standards` | Records, pattern matching, nullable types, async/await best practices |
| `concurrency` | async/await, Channel, Parallel.ForEachAsync, choosing the right abstraction |
| `di-patterns` | IServiceCollection extensions, lifetime management, keyed services |
| `configuration` | IOptions pattern, startup validation, strongly-typed config |

### Which skills each command loads

| Command | `spec` | `principles` | `testing` | `debug` | `done` | `decision` |
|---------|:---:|:---:|:---:|:---:|:---:|:---:|
| `/write` | if Spec Gate¹ | always | always | — | always | — |
| `/fix` | — | if design problem | always | always | always | — |
| `/review` | — | always | always | — | — | — |
| `/refactor` | — | if SOLID violation | always | — | always | — |
| `/decision` | — | — | — | — | — | standalone |

> ¹ **Spec Gate** — three questions before writing code: (1) Is this a bug fix or internal change? (2) Does a C# interface already exist? (3) Can you name 3+ boundary cases immediately? If all YES → skip spec, go straight to TDD. Any NO → load `spec` first.

## How It Works

**Two layers** — workflow skills provide step-by-step processes, methodology & domain skills provide knowledge.

- **Workflow skills** use numbered steps with explicit confirmation gates — Claude won't write code until you approve the plan
- **Methodology skills** are auto-loaded by workflows based on context (e.g., `/fix` always loads `testing`, optionally loads `principles` if the root cause is structural)
- **Domain skills** provide .NET MVC-specific patterns (architecture, EF Core, UnitOfWork, DI) and are loaded when the task touches those areas
- `disable-model-invocation: true` on workflow skills — no accidental auto-triggering

## Architecture

This plugin enforces a four-tier architecture:

```
{Project}.Web/          → Controllers, Views, Middleware
{Project}.Service/      → Business logic, ViewModels, Mapper
{Project}.Data/         → Entities, Repositories, UnitOfWork, EF Core
{Project}.Library/      → Shared helpers, enums, exceptions (no framework dependency)
```

**Dependency direction:** `Web → Service → Data → Library`

Key patterns: UnitOfWork, IDbContextFactory, ServiceResult\<T\>, soft delete, IAuthService.

## Workflow

```
/write "add user auth"
  → Spec Gate (interface defined? boundary cases clear?)
      YES → plan → confirm → TDD cycles → /review
      NO  → /spec (interface + invariants) → TDD cycles → /review

/fix "login crash"      →  diagnose  →  confirm  →  Red/Green/Refactor  →  /review
/refactor src/auth/     →  smell analysis  →  confirm  →  incremental transforms  →  /review
/decision "Dapper vs EF Core"  →  assumption audit  →  4-dimension scoring  →  pre-mortem  →  recommendation
```

## Specialized Agents

Agents are AI personas with deep domain expertise. They are invoked automatically when Claude detects relevant tasks.

| Agent | Expertise |
|-------|-----------|
| **efcore-migration-specialist** | Migration conflicts, schema evolution, Fluent API troubleshooting, Global Query Filter issues, seed data |
| **mvc-runtime-troubleshooter** | DI lifetime bugs (Scoped-in-Singleton, ObjectDisposedException), async/await deadlocks, DbContext threading, CancellationToken propagation, IDbContextFactory parallel queries, BackgroundService scope management |

## Suggested CLAUDE.md Snippet

Add this to your project's `CLAUDE.md` for consistent skill routing:

```markdown
# Agent Guidance: dotnet-mvc-skills

Routing (invoke by name)
- C# quality: csharp-coding-standards, concurrency
- Architecture: three-tier-architecture
- Data access: efcore-patterns, database-performance, unitofwork-dbcontextfactory
- DI / config: di-patterns, configuration
- Quality gate: run /review after completing each feature

Specialist agents (auto-invoked)
- efcore-migration-specialist, mvc-runtime-troubleshooter
```

## Credits

This plugin is built upon ideas and patterns from the following projects:

- **[bouob/coding-skills](https://github.com/bouob/coding-skills)** — TDD workflow structure, spec-driven design, and the two-layer skill architecture (workflow + methodology)
- **[Aaronontheweb/dotnet-skills](https://github.com/Aaronontheweb/dotnet-skills)** — .NET ecosystem skill patterns including EF Core, C# coding standards, database performance, concurrency, and DI/configuration best practices

## License

MIT
