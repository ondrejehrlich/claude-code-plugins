# Task Shredder

Shred complex tasks into parallel agent workflows with research, planning, and verification.

Task Shredder is a Claude Code command that acts as an **orchestrator** — it never writes code itself. Instead, it breaks your task into isolated phases, spawns dedicated sub-agents for each, enforces file ownership across parallel workers, and validates everything before calling it done.

## Usage

```
/task-shredder <task description>   # Start a new task
/task-shredder                      # List existing tasks or start new
/task-shredder resume               # Resume an interrupted task
```

## Recommended: Auto mode

Task Shredder spawns many sub-agents across all 7 steps. To avoid constant permission prompts, run Claude Code with auto-approve permissions:

```bash
claude --dangerously-skip-permissions
```

Or use [Auto Mode](https://claude.com/blog/auto-mode) for a safer alternative that still lets agents work uninterrupted while keeping guardrails in place.

> **Use at your own risk.** This grants Claude broad access to your filesystem and shell. Always work in a version-controlled repository so you can roll back via the git checkpoint tags if needed.

## How It Works

Every task goes through a 7-step pipeline. The orchestrator manages the lifecycle, tracks state in `task.json`, and keeps your context window clean by delegating all heavy lifting to sub-agents.

### Pipeline

```
[1/7] Research        Parallel codebase exploration (4 scout agents)
[2/7] Questions       Adaptive clarifying questions in batches of 3
[3/7] Planning        Implementation plan + automated validation
[4/7] Splitting       Phase decomposition with file ownership
[5/7] Implementation  Parallel agents per phase, wave-based execution
[6/7] Verification    Quality + functionality + plan-diff review
[7/7] Fixes           Findings routed back to original agents
```

### Step 1 — Research

Four parallel Explore agents scan your codebase simultaneously:

| Agent | Focus |
|-------|-------|
| Models & Database | Eloquent models, migrations, enums, relationships |
| Observers & Side Effects | Observers, jobs, notifications, event listeners |
| Business Logic & API | Actions, services, queries, controllers, routes |
| Frontend & Tests | Vue/Inertia components, existing test coverage |

All findings go into `research.md`. The orchestrator never reads this file — it stays out of the main context window.

### Step 2 — Questions

A question-generation agent reads the research and produces batches of 3 clarifying questions. Each question is presented one at a time via interactive dialog with selectable options (including a recommended choice).

After each batch, an evaluator decides if more questions are needed based on your answers. The loop continues until all critical decisions are covered — complex tasks may need multiple rounds.

### Step 3 — Planning

A planner agent reads the research + your Q&A decisions and writes a detailed `plan.md` with:
- Architecture decisions
- Step-by-step implementation with exact file paths and method signatures
- Database/API/frontend changes
- Test plan
- Requirements traceability (every Q&A decision and research risk mapped to a plan step)

A **plan-checker** agent then validates coverage — Q&A decisions, research risks, observer side effects, code reuse, test coverage, and file inventory. Gaps trigger automatic patching (up to 3 rounds). You review and approve the plan before proceeding.

### Step 4 — Splitting

A splitter agent divides the plan into 3-5 isolated phases following dependency order:

```
Phase 1 (Database) → Phase 2 (Models) → Phase 3 (Logic) → Phase 4 (Controllers) → Phase 5 (Frontend)
```

Key rules:
- **No file overlap** — no two phases touch the same file
- **Tests travel with code** — tests belong to the phase that creates the tested class
- **Waves pre-computed** — independent phases run in parallel

Produces `phases.md` (detailed, for agents) and `phases-summary.json` (compact, for the orchestrator).

### Step 5 — Implementation

Agents execute in waves based on the dependency graph. Before each wave, a git checkpoint tag is created for rollback safety.

- One agent per phase, running in parallel within each wave
- A **persistent code-reviewer** agent stays alive across all phases and reviews each one for file ownership compliance, code conventions, and obvious bugs
- Rejected phases get sent back to the original agent (which retains full context) for fixes
- Tests run after each wave completes on the full codebase
- If agents discover plan issues, they write to `issues.md` — blockers trigger automatic replanning

Commit format: `task(<slug>): <description>`

### Step 6 — Verification

Two agents run in parallel:

| Agent | Checks |
|-------|--------|
| **Quality** | PSR-12, architecture compliance, type safety, N+1 queries, security, DRY |
| **Functionality** | Test execution, observer side effects, edge cases, migration safety, **plan-diff** |

The plan-diff is a systematic checklist verifying every planned file, class, method, route, and test actually exists in the codebase. Produces a coverage percentage.

If zero critical findings and plan coverage >= 90%, you can skip the fixes phase entirely.

### Step 7 — Fixes

A router agent maps findings to their owning phases by file path. Findings are then sent back to the **original implementation agents** via `SendMessage` — they already have full context, so no re-reading research or plans.

Fixes are committed with format: `task(<slug>): fix <finding-id> - <description>`

A final test run confirms everything passes.

## State & Resume

All artifacts live in `.TaskShredder/<timestamp>-<slug>/`:

```
.TaskShredder/2026-03-25-14-30-payment-refund/
├── task.json                    # Full state with timestamps
├── research.md                  # Codebase exploration findings
├── questions-batch-1.md         # Question batches
├── questions-batch-2.md
├── plan.md                      # Implementation plan
├── plan-validation.md           # Validation report
├── phases.md                    # Detailed phase breakdown
├── phases-summary.json          # Compact phase data
├── issues.md                    # Issues found during implementation
├── verification-quality.md      # Quality review
├── verification-functionality.md # Functionality + plan-diff review
├── fix-assignments.json         # Findings mapped to phases
└── fixes-log.md                 # Consolidated fix results
```

Every step transition is timestamped in `task.json`. If interrupted, resume picks up where you left off:
- Completed steps are preserved
- The interrupted step restarts fresh
- Implementation resumes from the last incomplete phase
- Git checkpoint tags allow rollback to any wave boundary

## Model Selection

Agents use different model tiers to optimize for cost and speed:

| Agent | Model | Why |
|-------|-------|-----|
| Researcher | Opus | Deep codebase exploration needs reasoning |
| Question generators | Sonnet | Lightweight generation |
| Planner | Opus | Complex architectural decisions |
| Plan checker/patcher | Sonnet | Checklist validation |
| Splitter | Sonnet | Mechanical decomposition |
| Implementation agents | Opus | Code writing needs full reasoning |
| Code reviewer | Sonnet | Persistent, pattern-matching review |
| Verification agents | Opus | Deep analysis |
| Fix router/logger | Sonnet | Simple mapping and consolidation |

## Git Safety

- `.TaskShredder/` is automatically added to `.gitignore`
- Pre-wave checkpoint tags: `task-shredder/<slug>/pre-wave-<N>`
- Rollback available during resume (with user confirmation)
- All commits follow `task(<slug>):` convention for easy filtering