# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project purpose

This is a Rust workspace for **CLS 4.0** — a full rewrite of a 16-service .NET Core 3.1 LMS (Learning Management System) backend to Rust, using the Strangler-fig strategy. The frontend (Vue 2) and all API/JWT/RabbitMQ event contracts are kept unchanged throughout migration. No Rust code exists yet; the repo currently holds planning docs and will grow into a Cargo workspace.

## Planned workspace layout

```
cls-rust/
  Cargo.toml              ← workspace root
  crates/
    cls-core/             ← EventBus trait, JWT middleware, error types
    cls-db/               ← SQLx pool + Tiberius (SQL Server) wrapper
    cls-eventbus/         ← IEventBus trait + RabbitMQ impl (lapin)
    cls-auth/             ← JWT validation, claims parsing
    cls-observability/    ← tracing init, OpenTelemetry OTLP → SigNoz
  services/
    mailsender/           ← Phase 1 (first service)
    notificationservice/  ← Phase 2
    logservice/           ← Phase 2
    serverfiles/          ← Phase 3
    signalr/              ← Phase 3 (highest risk)
    ...
  docs/                   ← system docs (read these before implementing a service)
```

## Build & test commands

Once Cargo.toml exists:

```bash
rtk cargo build                          # build all
rtk cargo check                          # fast type-check
rtk cargo clippy                         # lint (required before commit)
rtk cargo test                           # all tests
cargo test -p mailsender                 # single service tests
cargo test -p mailsender specific_test   # single test
cargo fmt --check                        # format check (required before commit)
```

## Coding conventions

These are non-negotiable for all new Rust code:

- **Error handling**: `thiserror` in library crates (`crates/`), `anyhow` in service binaries (`services/`)
- **Async**: Tokio only — no mixing runtimes
- **Logging**: `tracing` macros (`info!`, `warn!`, `error!`) — never `println!`
- **Naming**: Rust `snake_case` internally; map to JSON `PascalCase` via `serde(rename_all = "PascalCase")` to match legacy .NET DTOs
- **Web framework**: Axum (built on Tokio/Hyper)
- **Database**: SQLx with compile-time query checking; Tiberius for SQL Server connections
- **Observability**: every service calls `cls_observability::init("service-name")` at the start of `main()`

## Key architectural constraints

**Never break API/event compatibility.** JSON DTO field names (PascalCase), JWT claims format, and RabbitMQ routing keys + payload schemas must stay identical to the .NET originals. The SQL Server and MongoDB schemas are owned by EF Core migrations — Rust services are consumers, not schema owners.

**Strangler-fig cutover flow per service:**
1. Implement Rust service alongside running .NET service (shared DB)
2. Run parity tests (same input → compare .NET output vs Rust output)
3. Canary traffic: 1% → 10% → 50% → 100% via Gateway feature flag
4. After 2 weeks stable: retire .NET image

## Migration phases (current status: Phase 0 — preparation)

| Phase | Services | Risk |
|---|---|---|
| 0 | Workspace setup, CI/CD, training | Low |
| 1 | MailSender (lapin + Apalis + lettre) | Low — validates Rust pattern |
| 2 | NotificationService (FCM), LogService (Mongo) | Low |
| 3 | ServerFiles, SignalR Hub | High — SignalR protocol is risky |
| 4 | SharedServices, SystemService, CommunicationService, ReportService | Low-Medium |
| 5 | TrainingRouteService, QuestionService, CourseService, UserService | Very High |
| 6 | IdentityServer4 → Keycloak | Medium |
| 7 | Ocelot Gateway → Pingora/Envoy | Medium |

**Decision gate G1** (after Phase 1): if Rust MailSender doesn't achieve ≥2× .NET in RAM or throughput, stop and evaluate upgrading to .NET 8 instead.

## Key docs to read before implementing a service

- `docs/02-backend-feature-catalog.md` — what each service must do
- `docs/services/<service-name>.md` — use cases, validators, business rules per service
- `docs/04-database-schema.md` — SQL Server + MongoDB schemas (Rust must match exactly)
- `docs/07-rust-migration-plan.md` — tech-stack mappings (.NET component → Rust equivalent), risk table

## C# → Rust crate mappings (quick reference)

| .NET | Rust |
|---|---|
| MediatR | Custom `CommandHandler<C,R>` trait + dispatcher |
| MediatR Behaviors | Tower middleware layers |
| FluentValidation | `validator` crate |
| Hangfire | `apalis` + `tokio-cron-scheduler` |
| Serilog | `tracing` + `tracing-subscriber` |
| Swashbuckle | `utoipa` |
| MongoDB.Driver | `mongodb` (official) |
| AutoMapper | `From`/`TryFrom` trait impls |
| Polly | `backoff` crate or Tower retry layer |

<!-- rtk-instructions v2 -->
# RTK (Rust Token Killer) - Token-Optimized Commands

## Golden Rule

**Always prefix commands with `rtk`**. If RTK has a dedicated filter, it uses it. If not, it passes through unchanged. This means RTK is always safe to use.

**Important**: Even in command chains with `&&`, use `rtk`:
```bash
# ❌ Wrong
git add . && git commit -m "msg" && git push

# ✅ Correct
rtk git add . && rtk git commit -m "msg" && rtk git push
```

## RTK Commands by Workflow

### Build & Compile (80-90% savings)
```bash
rtk cargo build         # Cargo build output
rtk cargo check         # Cargo check output
rtk cargo clippy        # Clippy warnings grouped by file (80%)
rtk tsc                 # TypeScript errors grouped by file/code (83%)
rtk lint                # ESLint/Biome violations grouped (84%)
rtk prettier --check    # Files needing format only (70%)
rtk next build          # Next.js build with route metrics (87%)
```

### Test (60-99% savings)
```bash
rtk cargo test          # Cargo test failures only (90%)
rtk go test             # Go test failures only (90%)
rtk jest                # Jest failures only (99.5%)
rtk vitest              # Vitest failures only (99.5%)
rtk playwright test     # Playwright failures only (94%)
rtk pytest              # Python test failures only (90%)
rtk rake test           # Ruby test failures only (90%)
rtk rspec               # RSpec test failures only (60%)
rtk test <cmd>          # Generic test wrapper - failures only
```

### Git (59-80% savings)
```bash
rtk git status          # Compact status
rtk git log             # Compact log (works with all git flags)
rtk git diff            # Compact diff (80%)
rtk git show            # Compact show (80%)
rtk git add             # Ultra-compact confirmations (59%)
rtk git commit          # Ultra-compact confirmations (59%)
rtk git push            # Ultra-compact confirmations
rtk git pull            # Ultra-compact confirmations
rtk git branch          # Compact branch list
rtk git fetch           # Compact fetch
rtk git stash           # Compact stash
rtk git worktree        # Compact worktree
```

Note: Git passthrough works for ALL subcommands, even those not explicitly listed.

### GitHub (26-87% savings)
```bash
rtk gh pr view <num>    # Compact PR view (87%)
rtk gh pr checks        # Compact PR checks (79%)
rtk gh run list         # Compact workflow runs (82%)
rtk gh issue list       # Compact issue list (80%)
rtk gh api              # Compact API responses (26%)
```

### JavaScript/TypeScript Tooling (70-90% savings)
```bash
rtk pnpm list           # Compact dependency tree (70%)
rtk pnpm outdated       # Compact outdated packages (80%)
rtk pnpm install        # Compact install output (90%)
rtk npm run <script>    # Compact npm script output
rtk npx <cmd>           # Compact npx command output
rtk prisma              # Prisma without ASCII art (88%)
```

### Files & Search (60-75% savings)
```bash
rtk ls <path>           # Tree format, compact (65%)
rtk read <file>         # Code reading with filtering (60%)
rtk grep <pattern>      # Search grouped by file (75%). Format flags (-c, -l, -L, -o, -Z) run raw.
rtk find <pattern>      # Find grouped by directory (70%)
```

### Analysis & Debug (70-90% savings)
```bash
rtk err <cmd>           # Filter errors only from any command
rtk log <file>          # Deduplicated logs with counts
rtk json <file>         # JSON structure without values
rtk deps                # Dependency overview
rtk env                 # Environment variables compact
rtk summary <cmd>       # Smart summary of command output
rtk diff                # Ultra-compact diffs
```

### Infrastructure (85% savings)
```bash
rtk docker ps           # Compact container list
rtk docker images       # Compact image list
rtk docker logs <c>     # Deduplicated logs
rtk kubectl get         # Compact resource list
rtk kubectl logs        # Deduplicated pod logs
```

### Network (65-70% savings)
```bash
rtk curl <url>          # Compact HTTP responses (70%)
rtk wget <url>          # Compact download output (65%)
```

### Meta Commands
```bash
rtk gain                # View token savings statistics
rtk gain --history      # View command history with savings
rtk discover            # Analyze Claude Code sessions for missed RTK usage
rtk proxy <cmd>         # Run command without filtering (for debugging)
rtk init                # Add RTK instructions to CLAUDE.md
rtk init --global       # Add RTK to ~/.claude/CLAUDE.md
```

## Token Savings Overview

| Category | Commands | Typical Savings |
|----------|----------|-----------------|
| Tests | vitest, playwright, cargo test | 90-99% |
| Build | next, tsc, lint, prettier | 70-87% |
| Git | status, log, diff, add, commit | 59-80% |
| GitHub | gh pr, gh run, gh issue | 26-87% |
| Package Managers | pnpm, npm, npx | 70-90% |
| Files | ls, read, grep, find | 60-75% |
| Infrastructure | docker, kubectl | 85% |
| Network | curl, wget | 65-70% |

Overall average: **60-90% token reduction** on common development operations.
<!-- /rtk-instructions -->
