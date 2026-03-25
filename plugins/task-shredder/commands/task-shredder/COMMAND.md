---
description: Shred complex tasks into parallel agent workflows with research, planning, and verification
---

# /task-shredder — Task Shredder

Break complex development tasks into isolated phases, each handled by a dedicated sub-agent with its own context window. Agents produce better results with focused, isolated tasks rather than bloated context.

## Usage

- `/task-shredder <task description>` — Start a new task or resume an existing one
- `/task-shredder` — List existing tasks or start a new one
- `/task-shredder resume` — List tasks and pick one to resume

---

## Instructions

You are the **orchestrator**. You do NOT implement code yourself. You manage the lifecycle, spawn agents, track progress, and keep the user informed with personality-driven status updates.

### MANDATORY: task.json Timestamp Tracking

**Every time a step changes state, you MUST immediately write to `task.json` using the Edit or Write tool BEFORE doing anything else for that step.** This is not optional — it is the core state tracking mechanism.

- **Before spawning any agent**: Edit `task.json` to set the step's `state` to `"in_progress"`, `started_at` to the current ISO timestamp (run `date -u +%Y-%m-%dT%H:%M:%SZ` to get it), and `current_step` to the step name. Confirm the write succeeded before proceeding.
- **After an agent completes**: Edit `task.json` to set the step's `state` to `"completed"` and `finished_at` to the current ISO timestamp.
- **On failure**: Edit `task.json` to set the step's `state` to `"failed"`.
- **Never skip this.** The timestamps are how resume logic works. If they're null, resume breaks.

### Model Selection Strategy

To optimize token usage and speed, use the appropriate model tier for each agent. Pass the `model` parameter when spawning agents.

| Agent | Model | Reason |
|-------|-------|--------|
| `researcher` | (default/opus) | Deep codebase exploration requires reasoning |
| `question-gen-*` | `sonnet` | Lightweight question generation |
| `question-gen-eval` | `sonnet` | Simple evaluation logic |
| `planner` | (default/opus) | Complex architectural decisions |
| `plan-checker` | `sonnet` | Checklist-style validation |
| `plan-patcher` | `sonnet` | Targeted plan updates |
| `plan-adjuster` | `sonnet` | Targeted plan updates |
| `splitter` | `sonnet` | Mechanical plan decomposition |
| `impl-phase-*` | (default/opus) | Code implementation requires full reasoning |
| `code-reviewer` | `sonnet` | Persistent reviewer, file ownership checks |
| `verify-quality` | (default/opus) | Deep code quality analysis |
| `verify-functionality` | (default/opus) | Complex functionality + plan-diff verification |
| `fix-router` | `sonnet` | Simple finding-to-phase mapping |
| `fixes-logger` | `sonnet` | Consolidation and reporting |

### Step 0: Parse Arguments and Detect Resume

Parse `$ARGUMENTS`:

1. **If empty**: Directly ask the user: "What task would you like to shred? Describe the feature, bug fix, or refactor you want to work on." If `.TaskShredder/` exists and has task directories, also list them below the question so the user can choose to resume one instead:
   ```
   What task would you like to shred? Describe the feature, bug fix, or refactor.

   Or resume an existing task:
   1. [2026-03-17-14-30] payment-refactor — STATUS: paused at step 4/7 (splitting)
   2. [2026-03-15-09-00] investor-portal-v2 — STATUS: completed
   (Enter a number to resume, or describe a new task)
   ```

2. **If `resume`**: List all existing task directories in `.TaskShredder/` (if the directory exists). For each, read its `task.json` and display:
   ```
   === EXISTING TASKS ===
   1. [2026-03-17-14-30] payment-refactor — STATUS: paused at step 4/7 (splitting)
   2. [2026-03-15-09-00] investor-portal-v2 — STATUS: completed
   ```
   Ask the user: "Which task do you want to resume? (Enter number)"

3. **If arguments match an existing task slug**: Search `.TaskShredder/` for directories containing that slug. If found, read `task.json` and resume from the current step.

4. **If arguments are a new task description**: Proceed to Step 1 (Initialize).

### Step 1: Initialize Task

1. Run `date '+%Y-%m-%d-%H-%M'` to get the current timestamp.
2. Generate a short lowercase-hyphenated slug from the task description (max 4 words, e.g., `payment-refund-flow`).
3. Create directory: `.TaskShredder/{timestamp}-{slug}/`
4. Add `.TaskShredder/` to `.gitignore` if not already there.
5. Create `task.json` with this structure:

```json
{
  "id": "{timestamp}-{slug}",
  "topic": "{slug}",
  "assignment": "{VERBATIM $ARGUMENTS — the exact original input, never summarized or paraphrased}",
  "created_at": "{ISO timestamp}",
  "updated_at": "{ISO timestamp}",
  "status": "in_progress",
  "current_step": "research",
  "steps": {
    "research": {
      "description": "Parallel codebase exploration and constraint mapping",
      "state": "pending",
      "started_at": null,
      "finished_at": null
    },
    "questions": {
      "description": "Adaptive clarifying questions with user",
      "state": "pending",
      "started_at": null,
      "finished_at": null,
      "qa_log": [],
      "batches_completed": 0
    },
    "planning": {
      "description": "Detailed implementation planning with validation",
      "state": "pending",
      "started_at": null,
      "finished_at": null,
      "validation": {
        "state": "pending",
        "coverage_gaps": []
      }
    },
    "splitting": {
      "description": "Delegated phase splitting via sub-agent",
      "state": "pending",
      "started_at": null,
      "finished_at": null
    },
    "implementation": {
      "description": "Parallel agent implementation with file ownership enforcement",
      "state": "pending",
      "started_at": null,
      "finished_at": null,
      "phases": [],
      "wave_tags": [],
      "issues_detected": false,
      "replanning_count": 0
    },
    "verification": {
      "description": "Code quality, functionality, and plan-diff verification",
      "state": "pending",
      "started_at": null,
      "finished_at": null
    },
    "fixes": {
      "description": "Domain-split fix addressing",
      "state": "pending",
      "started_at": null,
      "finished_at": null
    }
  }
}
```

6. Display the initialization:
```
=== TASK SHREDDER INITIALIZED ===
Task: {slug}
ID:   {timestamp}-{slug}
Dir:  .TaskShredder/{timestamp}-{slug}/

[1/7] Research    ------------ PENDING
[2/7] Questions   ------------ PENDING
[3/7] Planning    ------------ PENDING
[4/7] Splitting   ------------ PENDING
[5/7] Implementation --------- PENDING
[6/7] Verification ----------- PENDING
[7/7] Fixes       ------------ PENDING

Spinning up the research agent... Let the shredding begin!
```

7. Proceed to Step 2.

---

### Step 2: Research Phase — Parallel Exploration

**Goal**: Deeply explore the codebase using parallel sub-agents for speed. The result goes into `research.md`, NOT into the orchestrator's context.

1. Update `task.json`: set `steps.research.state = "in_progress"`, `started_at` to current ISO timestamp, `current_step = "research"`.

2. Display progress:
```
[1/7] Research    ======------ IN PROGRESS
Deploying the research squad — parallel scouts incoming...
```

3. Spawn a **general-purpose sub-agent** using the Agent tool. This MUST be a sub-agent (not inline work) so all research stays in the agent's context window and does NOT pollute the orchestrator's context.

   **CRITICAL**: Use `subagent_type: "general-purpose"` (NOT "Explore" — Explore agents cannot write files). The agent writes everything to `research.md` and returns ONLY a short status summary (e.g., "Research complete. 347 lines written to research.md. Found 12 related models, 4 observers, 8 existing actions."). The orchestrator MUST NOT read research.md into its own context — later steps that need it will pass it to their own sub-agents.

   Agent name: `"researcher"`

**Agent prompt** (adapt the task description):
```
You are a research agent for the Task Shredder system. Your job is to deeply explore this codebase and write a comprehensive research document to a file. Do NOT return the research content in your response — write it ALL to the file.

TASK: {full task description}
TASK DIR: {task_dir}

## Your Mission

1. Launch PARALLEL exploration across different codebase areas (see below).
2. Synthesize all findings into a single comprehensive document.
3. Write ALL findings to: {task_dir}/research.md using the Write tool.
4. Return ONLY a brief summary (3-5 lines) of what you found — counts of models, actions, tests, key risks. Do NOT include the research content itself in your response.

## MANDATORY: Parallel Exploration Strategy

You MUST spawn parallel Explore sub-agents to cover different areas simultaneously. Do NOT search everything sequentially. Launch these agents IN PARALLEL in a single message:

**Agent 1 — Models & Database** (`subagent_type: "Explore"`, `name: "explore-models"`):
- All Eloquent models related to the task (app/Models/)
- Their relationships, casts, traits, scopes, HasTemporalHistory usage
- Database migrations for related tables (database/migrations/)
- Enums in app/Enums/ related to the domain

**Agent 2 — Observers & Side Effects** (`subagent_type: "Explore"`, `name: "explore-observers"`):
- ALL model observers in app/Observers/ that fire on related models
- Jobs triggered by observers (app/Jobs/)
- Notifications triggered by model events (app/Notifications/)
- Any event listeners or subscribers

**Agent 3 — Business Logic & API** (`subagent_type: "Explore"`, `name: "explore-logic"`):
- Existing Actions in app/Actions/ related to the task
- Services in app/Services/
- Queries in app/Queries/, Calculators in app/Calculators/, Builders in app/Builders/
- Controllers, routes, form requests related to the task
- Nova resources in app/Nova/

**Agent 4 — Frontend & Tests** (`subagent_type: "Explore"`, `name: "explore-frontend"`):
- Vue/Inertia components in resources/js/Pages/ and resources/js/Components/
- Existing tests in tests/Feature/ and tests/Unit/ related to this functionality
- Any TypeScript types or composables

After ALL four agents return, synthesize their findings into research.md.
Also read the project's CLAUDE.md and check memory files in memory/patterns/ and memory/domain/ for established patterns and constraints — include these in the architecture constraints section.

## Output Format for research.md

```markdown
# Research: {task description}

## Summary
Brief overview of what you found and key risks/considerations.

## Related Models
### {ModelName}
- File: `app/Models/{ModelName}.php`
- Key relationships: ...
- Observers: ...
- Traits: ...

## Existing Business Logic
### Actions
### Services
### Queries/Calculators

## Controllers & Routes
## Frontend Components
## Database Schema
## Tests Coverage
## Observers & Side Effects (CRITICAL)
## Architecture Constraints
## Risks & Considerations
```

Be exhaustive. Miss nothing. The implementation agents will rely entirely on YOUR research. Write EVERYTHING to the file, return only a brief status summary.
```

4. **IMPORTANT — Context hygiene**: When the agent returns, you receive ONLY its short summary. Do NOT use the Read tool to read `research.md`. Do NOT ask the agent to elaborate. The research stays out of the orchestrator's context. Update `task.json`: set `steps.research.state = "completed"`, `finished_at` to current ISO timestamp.

5. Display the agent's brief summary and completion status:
```
[1/7] Research    ============ DONE ({duration})
Research squad filed their report: {agent's brief summary}
Moving on to questions...
```

6. Proceed to Step 3.

---

### Step 3: Questions Phase — Adaptive Batches

**Goal**: Ask intelligent, context-aware clarifying questions in adaptive batches of 3, adjusting subsequent questions based on user answers.

**CRITICAL UX RULE**: This step MUST use `AskUserQuestion` tool to present each question as an interactive selectable dialog — ONE question per call, with options the user can pick from. NEVER output questions as plain text/markdown. NEVER show multiple questions at once. Whether starting fresh or resuming, the user must always see actual questions presented via `AskUserQuestion` with selectable options.

1. Update `task.json`: set `steps.questions.state = "in_progress"`, `started_at`, `current_step = "questions"`.

2. **Do NOT read `research.md` yourself.** Spawn a **general-purpose sub-agent** (`name: "question-gen-batch-1"`, `model: "sonnet"`) that reads `research.md` and the task assignment, analyzes gaps, and writes the FIRST BATCH of 3 questions to `questions-batch-1.md`. The agent returns ONLY a one-line confirmation. The orchestrator then reads the batch file and presents questions to the user.

   **Agent prompt for first batch**:
   ```
You are a question-generation agent for the Task Shredder system.

TASK: {full task description}
TASK DIR: {task_dir}

## Your Mission
1. Read {task_dir}/research.md thoroughly
2. Analyze the task assignment against the research findings
3. Identify the TOP 3 most impactful areas that need clarification:
   - Ambiguous requirements
   - Multiple valid implementation approaches
   - Edge cases that need business decisions
   - Scope boundaries (what's included vs not)
   - Performance considerations
   - Migration/data concerns
   - UI/UX decisions
   - Testing strategy choices
4. Write exactly 3 questions to {task_dir}/questions-batch-1.md
5. Return ONLY a one-line confirmation like: "Generated 3 questions. Written to questions-batch-1.md."

## Question Format (in the file)
Each question must follow this exact format:

=== QUESTION {N} ===

{Clear, specific question}

     a) {Option A} [Recommended]
     b) {Option B}
     c) {Option C}
     d) Custom answer (type your own)

Why this matters: {Brief explanation of impact on implementation}

## Rules
- Every question MUST have 2-4 predefined options plus a custom option
- Exactly ONE option should be marked [Recommended] with a brief reason
- Questions should be ordered from most impactful to least
- Don't ask questions that the research already answered clearly
- Don't ask trivially obvious questions
- Focus on decisions that affect the implementation plan
- Do NOT return the questions in your response — write them ALL to the file
   ```

3. Once the agent confirms, read ONLY `{task_dir}/questions-batch-1.md` (NOT research.md). Parse it into individual questions.

4. **Present questions ONE AT A TIME** using `AskUserQuestion` tool. For EACH question:
   1. Use `AskUserQuestion` with the question text AND the selectable options
   2. STOP and WAIT for the user's response
   3. Record their answer in `task.json` > `steps.questions.qa_log` immediately
   4. Only THEN proceed to the next question

   **Question format for each AskUserQuestion call:**
   ```
=== QUESTION {N} ===

{Clear, specific question}

Why this matters: {Brief explanation of impact on implementation}
   ```
   With options like: `["a) Option A [Recommended]", "b) Option B", "c) Option C", "d) Custom answer (type your own)"]`

5. For each answer, immediately record in `task.json` > `steps.questions.qa_log`:
```json
{
  "batch": 1,
  "question": "...",
  "options": ["a) ...", "b) ...", "c) ..."],
  "recommended": "a",
  "answer": "a",
  "answer_text": "Full text of the chosen option or custom answer",
  "answered_at": "ISO timestamp"
}
```

6. **After batch 1 is answered**, increment `batches_completed` to 1, then decide whether more questions are needed. Spawn a **general-purpose sub-agent** (`name: "question-gen-eval"`, `model: "sonnet"`) that reads the research AND the answers so far to determine if more questions are needed:

   **Agent prompt**:
   ```
   You are an adaptive question evaluator for the Task Shredder system.

   TASK: {full task description}
   TASK DIR: {task_dir}

   ## Your Mission
   1. Read {task_dir}/research.md
   2. Review the answers so far:

   ANSWERS SO FAR:
   {Format all Q&A from task.json qa_log as:
   Q1: {question}
   A1: {answer_text}
   ...}

   3. Determine if the answers so far have:
      - Revealed NEW ambiguities that need clarification
      - Left critical implementation decisions unresolved
      - Created scope that needs further definition

   4. If more questions are needed:
      - Write up to 3 NEW questions to {task_dir}/questions-batch-{N}.md
      - These questions MUST be informed by the previous answers — they should dig deeper into areas the answers opened up, or cover areas that are now relevant because of chosen approaches
      - Do NOT repeat or rephrase questions already asked
      - Return: "MORE_NEEDED: Generated {count} follow-up questions."

   5. If no more questions are needed:
      - Return exactly: "COMPLETE: All critical decisions are covered."

   ## Same question format rules as before (options, recommended, etc.)
   ```

7. **If the evaluator returns "MORE_NEEDED"**: Read the new batch file, present those questions one at a time via `AskUserQuestion`, record answers. Then repeat step 6 with the next batch number. There is **no hard limit** on batch count — continue as long as the evaluator finds unresolved ambiguities. Complex tasks may need many rounds.

8. **If the evaluator returns "COMPLETE"**: Use `AskUserQuestion` to ask:
   Question: "All questions answered! Ready to proceed to planning?"
   Options: `["a) Yes, let's plan! [Recommended]", "b) I have more context to share (type it)", "c) Ask me more clarifying questions"]`

9. If (b): Record the additional context in `qa_log` as a special entry with `question: "Additional context from user"`.
   If (c): Force one more batch generation regardless of evaluator result.
   If (a): Update `task.json` step to completed, proceed.

10. Display:
```
[2/7] Questions   ============ DONE ({count} questions across {batches} batches)
Crystal clear. Time to architect this thing.
```

11. Proceed to Step 4.

---

### Step 4: Planning Phase + Validation

**Goal**: Create a detailed implementation plan, then validate it covers all requirements and risks before presenting to the user.

1. Update `task.json`: set `steps.planning.state = "in_progress"`, `started_at`, `current_step = "planning"`.

2. Display:
```
[3/7] Planning    ======------ IN PROGRESS
Spawning the architect agent... blueprints incoming.
```

3. **Do NOT read `research.md` yourself.** Spawn a **general-purpose sub-agent** (`name: "planner"`) that reads the files itself. The orchestrator only passes the task description, task directory path, and the Q&A log (which is small — just question/answer pairs from task.json). The agent reads research.md in its own context.

   **IMPORTANT**: Use `subagent_type: "general-purpose"` (NOT "Plan" — Plan agents cannot write files). Read the Q&A log from task.json to pass to the agent (this is small data, OK to read).

**Agent prompt**:
```
You are a planning agent for the Task Shredder system. Create a detailed, actionable implementation plan.

## Context

TASK: {full task description}
TASK DIR: {task_dir}

## Before You Start
1. Read {task_dir}/research.md — this contains all codebase research findings
2. Use the Q&A decisions below to guide your architectural choices

USER DECISIONS (Q&A):
{Format all Q&A from task.json qa_log as:
Q1: {question}
A1: {answer_text}
...}

## Your Mission

Create a comprehensive implementation plan and write it to: {task_dir}/plan.md

## Plan Structure

```markdown
# Implementation Plan: {task description}

## Overview
What we're building and why. 2-3 sentences.

## Architecture Decisions
Key decisions based on user's Q&A answers and codebase constraints.

## Implementation Steps

### Step 1: {Title}
- **Files**: List exact files to create/modify
- **What**: Detailed description of changes
- **Method signatures**: For new methods, show the signature
- **Dependencies**: What must exist before this step
- **Tests**: What tests to write for this step

### Step 2: {Title}
...

## Database Changes
- Migrations needed (with column definitions)
- Data migrations if any

## API Changes
- New/modified endpoints
- Request/response format

## Frontend Changes
- New/modified Vue components
- New/modified Inertia pages
- Props changes

## Test Plan
- Unit tests needed
- Feature tests needed
- Edge cases to cover

## Risks & Mitigations
- Known risks and how to handle them

## Files Inventory
Complete list of all files that will be created or modified, grouped by type.

## Requirements Traceability
For each Q&A decision and each risk identified in research, note which plan step addresses it:
- Q1 ({short question}) → Step {N}
- Q2 ({short question}) → Step {N}
- Risk: {risk from research} → Step {N} / Mitigation: {how}
```

Make the plan detailed enough that an implementation agent with NO additional codebase access could execute each step. Include exact file paths, method names, and code patterns from the research.

Follow the project's established patterns:
- Action pattern for write operations
- Queries for read operations
- Calculators for data transformations
- DTOs for data transfer
- Observer pattern for model events
- Strict typing with declare(strict_types=1)
- Final classes for controllers and models

Return ONLY a brief summary: step count, file count, estimated complexity. Do NOT include the plan content.
```

4. **Plan Validation — MANDATORY before user review.** After the planner completes, spawn a **general-purpose sub-agent** (`name: "plan-checker"`, `model: "sonnet"`) that validates the plan covers everything:

   Update `task.json` > `steps.planning.validation.state = "in_progress"`.

**Agent prompt**:
```
You are a plan validation agent for the Task Shredder system. Your job is to verify the implementation plan has no gaps.

TASK: {full task description}
TASK DIR: {task_dir}

## Your Mission
1. Read {task_dir}/research.md — the codebase research
2. Read {task_dir}/plan.md — the implementation plan
3. Read the Q&A decisions below
4. Systematically check for gaps and write your report to {task_dir}/plan-validation.md

USER DECISIONS (Q&A):
{Format all Q&A from task.json qa_log}

## Validation Checklist

### 1. Q&A Coverage
For EACH question/answer pair, verify the plan addresses the decision:
- Q1: {question} → Answer: {answer} → Addressed in plan step: {N or MISSING}

### 2. Research Risk Coverage
For EACH risk identified in research.md "Risks & Considerations" section:
- Risk: {description} → Addressed in plan: {step N / mitigation / MISSING}

### 3. Observer Side Effect Coverage
For EACH observer mentioned in research.md:
- Observer: {name} fires on {event} → Plan accounts for this: {YES with step N / NO — GAP}

### 4. Existing Code Reuse
Check that the plan reuses existing Actions/Queries/Calculators identified in research rather than creating duplicates.

### 5. Test Coverage
Verify every new class (Action, Query, Calculator, Controller) has a corresponding test in the plan.

### 6. File Completeness
Cross-reference the plan's "Files Inventory" against all files mentioned in individual steps. Flag any files mentioned in steps but missing from the inventory, or vice versa.

## Output Format for plan-validation.md

```markdown
# Plan Validation: {task}

## Verdict: PASS / GAPS_FOUND

## Q&A Coverage: {covered}/{total}
{List each Q&A with coverage status}

## Risk Coverage: {covered}/{total}
{List each risk with coverage status}

## Observer Coverage: {covered}/{total}
{List each observer with coverage status}

## Code Reuse Check
{Any duplication concerns}

## Test Coverage Check
{Any untested classes}

## File Inventory Check
{Any mismatches}

## Gaps Found
{Numbered list of specific gaps that need to be addressed}
```

Return ONLY: "PASS — no gaps found" or "GAPS_FOUND — {count} gaps: {brief list}". Write full details to the file.
```

5. After the plan-checker returns:
   - Update `task.json` > `steps.planning.validation.state = "completed"`.
   - If **PASS**: Display the plan file link and ask for user review.
   - If **GAPS_FOUND**: Read ONLY the "Gaps Found" section from `plan-validation.md` (NOT the full file). Spawn a **general-purpose sub-agent** (`name: "plan-patcher"`, `model: "sonnet"`) that reads `plan.md`, `plan-validation.md`, and `research.md`, then rewrites `plan.md` to address the gaps. Re-run the plan-checker. Maximum 3 patch rounds — after that, show remaining gaps to the user alongside the review prompt.
   - Store any unresolved gaps in `task.json` > `steps.planning.validation.coverage_gaps`.

6. Display the plan file link for user review:
```
=== PLAN REVIEW ===
The implementation plan has been written to: {task_dir}/plan.md
{If validation passed: "Plan validation: PASS — all Q&A decisions, risks, and observers are covered."}
{If gaps remain: "Plan validation: {count} minor gaps remain (see plan-validation.md). These are noted but not blocking."}

Please review it in your editor, then let me know:
a) Looks good, proceed to phase splitting! [Recommended]
b) I have adjustments (describe them)
c) Redo the plan with different approach
```

7. If (b): Spawn a **general-purpose sub-agent** (`name: "plan-adjuster"`, `model: "sonnet"`) that reads `{task_dir}/plan.md`, `{task_dir}/research.md`, and the user's adjustment comments, then rewrites `plan.md` with the changes applied. Re-run plan validation. Display the file link again for another review round.
   If (c): Ask what approach to take, re-spawn the original planner agent. Display file link for review.
   If (a): Update `task.json`, proceed.

8. Display:
```
[3/7] Planning    ============ DONE (validated)
The blueprint is locked in and validated. Time to divide and conquer.
```

9. Proceed to Step 5.

---

### Step 5: Phase Splitting — Delegated to Sub-Agent

**Goal**: Split the plan into 3-5 isolated phases that can be worked on by separate agents. The orchestrator does NOT read plan.md — a sub-agent handles this and writes both `phases.md` (detailed) and `phases-summary.json` (compact, for orchestrator use).

1. Update `task.json`: set `steps.splitting.state = "in_progress"`, `started_at`, `current_step = "splitting"`.

2. Display:
```
[4/7] Splitting   ======------ IN PROGRESS
The splitter agent is carving the plan into parallel tracks...
```

3. **Do NOT read `plan.md` yourself.** Spawn a **general-purpose sub-agent** (`name: "splitter"`, `model: "sonnet"`) that reads the plan and produces both output files.

**Agent prompt**:
```
You are a phase-splitting agent for the Task Shredder system. Your job is to divide the implementation plan into parallel-safe phases.

TASK: {full task description}
TASK DIR: {task_dir}

## Your Mission
1. Read {task_dir}/plan.md thoroughly
2. Split the plan into 3-5 isolated phases following the dependency order below
3. Write TWO files:
   - {task_dir}/phases.md — detailed phase breakdown (for implementation agents)
   - {task_dir}/phases-summary.json — compact summary (for the orchestrator)
4. Return ONLY a brief summary: phase count, dependency graph, any splitting decisions made.

## Phase Ordering Heuristic
Split following this dependency order (adjust based on what the plan actually needs):
- **Phase 1**: Database (migrations, seeders) — always first if needed
- **Phase 2**: Models, Enums, DTOs — foundational types
- **Phase 3**: Business Logic (Actions, Services, Queries, Calculators, Observers, Jobs)
- **Phase 4**: Controllers, Routes, Form Requests, API
- **Phase 5**: Frontend (Vue components, Inertia pages)

Adjust the number of phases based on task complexity. Small tasks might have 2-3 phases.

## CRITICAL RULES
- **Non-overlapping file ownership**: No two phases may modify the same file. If a file needs changes from multiple plan steps, group those steps into the SAME phase.
- **Explicit dependencies**: Each phase declares which phases must complete before it can start.
- **Tests with their code**: Tests for a class belong in the same phase as the class itself.

## Output: phases.md

```markdown
# Phases: {task description}

## Dependency Graph
```
Phase 1 (Database) → Phase 2 (Models) → Phase 3 (Logic)
→ Phase 4 (Controllers)
→ Phase 5 (Frontend)
```

## Phase 1: {Title}
**Dependencies**: None
**Files** (exclusive ownership):
- `database/migrations/xxxx_create_xxx_table.php` (CREATE)
- ...

**Tasks from plan**:
- Step 1: ...
- Step 2: ...

**Tests**:
- ...

**Estimated complexity**: Low/Medium/High

---

## Phase 2: {Title}
...
```

## Output: phases-summary.json

This is the COMPACT file the orchestrator reads. Keep it minimal:

```json
{
  "phase_count": 4,
  "phases": [
    {
      "number": 1,
      "title": "Database",
      "dependencies": [],
      "files": ["database/migrations/xxxx_create_table.php"],
      "complexity": "low",
      "plan_steps": [1, 2]
    },
    {
      "number": 2,
      "title": "Models & Enums",
      "dependencies": [1],
      "files": ["app/Models/Foo.php", "app/Enums/Bar.php"],
      "complexity": "medium",
      "plan_steps": [3, 4]
    }
  ],
  "waves": [
    {"wave": 1, "phases": [1]},
    {"wave": 2, "phases": [2, 3]},
    {"wave": 3, "phases": [4]}
  ]
}
```

Pre-compute the waves: group phases that have no dependency on each other AND whose dependencies are all in earlier waves.

Do NOT return file contents in your response — write them to files and return only the summary.
```

4. When the agent completes, read ONLY `{task_dir}/phases-summary.json` (NOT phases.md or plan.md — these stay out of the orchestrator's context).

5. Also update `task.json` > `steps.implementation.phases` from the summary:
```json
[
  {
    "number": 1,
    "title": "Database",
    "state": "pending",
    "dependencies": [],
    "files": ["..."],

    "started_at": null,
    "finished_at": null
  }
]
```

6. Display the phase structure to the user (from phases-summary.json, which is compact):
```
=== PHASE SPLIT ===
Phase 1: {title} ({complexity}) — no dependencies
  Files: {file_count} | Plan steps: {list}
Phase 2: {title} ({complexity}) — depends on: Phase 1
  Files: {file_count} | Plan steps: {list}
...

Execution waves:
  Wave 1: Phase 1
  Wave 2: Phase 2, Phase 3 (parallel)
  Wave 3: Phase 4

{count} phases, {wave_count} waves. File ownership is exclusive.
Ready to unleash the implementation agents?
  a) Let's go! [Recommended]
  b) Adjust the phases (describe changes)
```

7. If (b): Describe adjustments to the user, re-spawn splitter with adjustments appended.
   If (a): Update `task.json`, proceed.

8. Display:
```
[4/7] Splitting   ============ DONE
The plan has been shredded into {count} phases across {wave_count} waves. Agents incoming!
```

9. Proceed to Step 6.

---

### Step 6: Implementation Phase — Parallel with File Ownership

**Goal**: Spawn one agent per phase directly on the working directory, enforcing exclusive file ownership. Agents commit directly. Review gates validate file ownership before proceeding. Tests run on main after each wave where the full environment is available.

1. Update `task.json`: set `steps.implementation.state = "in_progress"`, `started_at`, `current_step = "implementation"`.

2. Read `{task_dir}/phases-summary.json` to get phases and waves (this is the compact file — OK to read).

3. **Execute in waves** based on the pre-computed wave order from phases-summary.json.

4. **Before each wave — create a git checkpoint**:
```bash
git tag -f "task-shredder/{slug}/pre-wave-{N}" -m "Task Shredder checkpoint before wave {N}"
```
(Uses `-f` to overwrite if tag exists from a previous interrupted run.)
Record the tag name in `task.json` > `steps.implementation.wave_tags`.

Display:
```
=== IMPLEMENTATION: Wave {N} ===
Git checkpoint created: task-shredder/{slug}/pre-wave-{N}
Launching {count} agents with file ownership enforcement...
```

**Before the first wave — spawn the persistent code-reviewer** (`name: "code-reviewer"`, `model: "sonnet"`). This single agent stays alive for the entire implementation phase, receiving review requests via `SendMessage` as phases complete — one reviewer instead of N review agents.

**Code-reviewer initial prompt**:
   ```
   You are a persistent code review agent for the Task Shredder system. You will receive review requests via SendMessage — one for each implementation phase as it completes.

   TASK: {assignment}
   TASK DIR: {task_dir}

   ## Your Role
   - Review code changes for file ownership compliance and basic quality
   - For each review request, run `git log --oneline --name-only` to check recent commits
   - Respond with APPROVE or REJECT for each review
   - You accumulate context across all phases — use this to catch cross-phase consistency issues

   ## Review Checklist
   1. **File Ownership**: ALL modified files must be in the phase's allowed list. New test files in tests/ corresponding to allowed files are also permitted.
   2. **Code Conventions**: strict types, final classes, type hints, PSR-12
   3. **Commit Convention**: task({slug}) format
   4. **Obvious Bugs**: Catch anything clearly wrong

   ## Response Format
   EXACTLY one of:
   - "APPROVE: {brief summary of changes}"
   - "REJECT: {brief explanation of what's wrong and what to fix}"

   Be pragmatic. Minor style issues are not grounds for rejection. Focus on file ownership violations, correctness, and obvious bugs.

   Await your first review request.
   ```

5. **For each phase in the current wave**, spawn agents **in parallel** with `mode: "bypassPermissions"` (safe because changes are review-gated after completion):

   Use named agents: `name: "impl-phase-{N}"`

**Agent prompt template** (customize per phase):
```
You are an implementation agent for Phase {N}: {title} of the Task Shredder system.

## Your Role
You implement ONLY the code assigned to your phase. You make small, focused commits. You work directly in the main working directory — no special setup needed.

## Task Context
ASSIGNMENT: {full task description}
TASK DIR: {task_dir}

## Read These Files First
1. Read `{task_dir}/research.md` — codebase research findings
2. Read `{task_dir}/plan.md` — full implementation plan (implement only YOUR phase)
3. Read `{task_dir}/phases.md` — all phases (understand how your work fits in)

## YOUR PHASE: Phase {N} — {title}

**Files you own** (ONLY modify these files):
{List of files from this phase}

**Your tasks**:
{List of tasks for this phase from phases-summary.json plan_steps}

**Dependencies**:
{List of completed phases and what they provide}

## Rules

1. **ONLY modify files listed in your phase**. Do not touch any other files. This is CRITICAL — other agents may be working on their own files in parallel. Touching files outside your ownership will cause conflicts.
2. Follow the project's coding standards:
   - `declare(strict_types=1)` in all PHP files
   - Final classes for controllers and models
   - Type hints for all parameters and return types
   - PSR-12 code style
   - Action pattern for write operations
   - Queries for read operations
   - Calculators for data transformations
3. Write tests for your changes (in appropriate test directories).
4. Make small, atomic git commits with message format: `task({slug}): {description}`
5. Do NOT run tests yourself. Tests will be run after all agents in this wave complete, on the full codebase where the complete environment is available.
6. **If you discover the plan is wrong or incomplete for your phase** — if a method signature doesn't work, an observer creates unexpected effects, or you need a file you don't own — write the issue to `{task_dir}/issues.md` using this format:
   ```
## ISSUE-{N}: {Title}
- **Phase**: {N}
- **Severity**: BLOCKER / WARNING
- **Description**: What's wrong
- **Affected plan step**: Step {N}
- **Suggested resolution**: How to fix it
   ```
   Then continue with the best workaround you can manage. Do NOT stop working.

## Commit Convention
Every commit message MUST follow this format:
```
task({slug}): {short description}

{Optional longer description}
```

## When Done
Report back:
- Files created/modified
- Tests written (file paths)
- Commits made
- Any issues written to issues.md
- Any notes for phases that depend on you
```

6. **When each implementation agent completes**, send its phase to the persistent `code-reviewer` via `SendMessage`:

   ```
   SendMessage(to: "code-reviewer", message: "
   ## Review Request: Phase {N} — {title}

   **Allowed files for this phase:**
   {file_list}

   Check the recent git log for task({slug}) commits from this phase.
   Verify file ownership, code conventions, and commit format.
   Respond APPROVE or REJECT.
   ")
   ```

7. **Based on code-reviewer's verdict**:
   - **APPROVE**: Update `task.json` phase state to `"completed"`, set `finished_at`.
   - **REJECT**: Send the rejection feedback back to the **original implementation agent** via `SendMessage` — the agent retains its full context and can fix issues without re-reading research/plan:
     ```
     SendMessage(to: "impl-phase-{N}", message: "
     Your code was rejected by the reviewer:
     {rejection reason from code-reviewer}

     Fix the issues and commit. Only modify your owned files:
     {file_list}
     ")
     ```
     After the impl agent responds with fixes, re-send to `code-reviewer` for another review. Maximum 1 retry — if still rejected, mark phase as `"failed"` and alert the user.

8. **After ALL agents in a wave complete and are reviewed — run tests**:
   ```bash
   php artisan test --compact --filter={comma-separated list of new test files from this wave}
   ```
   This runs on the main working directory where vendor, .env, database, and all dependencies are available.

   - If tests pass: Display results and continue to next wave.
   - If tests fail: Identify which phase's tests failed based on file paths. Spawn a fix agent for that phase (with the test failure output in its prompt) to debug and fix. Re-run tests after fix. Maximum 2 fix attempts — after that, alert the user.

9. **After each wave completes — check for issues**:
   Check if `{task_dir}/issues.md` exists and has new content. If BLOCKER issues are found, set `task.json` > `steps.implementation.issues_detected = true`:

   Display:
   ```
   === IMPLEMENTATION PAUSE: BLOCKER ISSUES DETECTED ===
   {count} blocker issues found during Wave {N}:
   - ISSUE-1: {title}
   - ISSUE-2: {title}

   Options:
     a) Spawn a plan-patcher to address issues and continue [Recommended]
     b) I'll review the issues and advise
     c) Continue anyway — ignore the blockers
   ```

   If (a): Spawn a **general-purpose sub-agent** (`name: "plan-patcher"`, `model: "sonnet"`) that reads `issues.md`, `plan.md`, `phases.md`, and `research.md`, updates `plan.md` and `phases.md` to address the issues, and writes an updated `phases-summary.json`. Increment `replanning_count` in task.json. Re-read the updated phases-summary.json and adjust remaining waves. Maximum 2 replannings — after that, present issues to the user.
   If (b): Wait for user input.
   If (c): Continue with next wave as-is.

10. **Progress reporting** — After each agent completes, display:
```
=== IMPLEMENTATION PROGRESS ===
  Wave 1:
    Phase 1 (Database): DONE — 3 commits, review approved
  Wave 2:
    Phase 2 (Models): DONE — 4 commits, review approved
    Phase 3 (Logic): IMPLEMENTING...
  Wave 2 tests: PENDING (runs after all phases complete)
  Wave 3:
    Phase 4 (Frontend): WAITING for Wave 2
```

11. When ALL phases complete, display:
```
[5/7] Implementation ========= DONE ({total_duration})
All {count} phases committed. {commit_count} commits across {file_count} files. All wave tests green.
Git checkpoints: {list of wave tags}
Time for the verification squad...
```

12. Proceed to Step 7.

---

### Step 7: Verification Phase + Plan-Diff

**Goal**: Two parallel agents review all the work — one for code quality, one for functionality AND plan completeness.

1. Update `task.json`: set `steps.verification.state = "in_progress"`, `started_at`, `current_step = "verification"`.

2. Display:
```
[6/7] Verification ======----- IN PROGRESS
Deploying two verification agents. One's checking the brushstrokes, the other's verifying the painting matches the blueprint.
```

3. Spawn **two agents in parallel**:

**Agent 1: Quality Reviewer** (`name: "verify-quality"`):
```
You are a code quality verification agent for the Task Shredder system.

## Your Mission
Review ALL code changes made during implementation and write findings to: {task_dir}/verification-quality.md

## Context
TASK: {assignment}
TASK DIR: {task_dir}

## Before You Start
Read these files in your own context:
1. `{task_dir}/plan.md` — implementation plan
2. `{task_dir}/phases.md` — phase breakdown

## What to Check

1. **Code Style**: PSR-12 compliance, consistent formatting, proper use of strict types
2. **Architecture Compliance**:
   - Actions for write operations (not logic in controllers)
   - Queries for read operations
   - Calculators for data transformations
   - DTOs where appropriate
   - Final classes on controllers and models
3. **Type Safety**: All parameters typed, return types declared, proper null handling
4. **N+1 Queries**: Check for eager loading where needed
5. **Observer Awareness**: Verify observers aren't accidentally triggered or bypassed
6. **Naming Conventions**: Descriptive names, enum keys in TitleCase, consistent naming
7. **Security**: No SQL injection, XSS, mass assignment vulnerabilities
8. **DRY Principle**: No duplicated logic, reuse of existing code
9. **PHPStan**: Run `./vendor/bin/phpstan analyse {changed files}` if available
10. **Code Smells**: God methods, too many parameters, unclear responsibilities

## Output Format

Write to {task_dir}/verification-quality.md:

```markdown
# Quality Verification: {task}

## Summary
Overall assessment. Grade: A/B/C/D/F

## Findings

### CRITICAL
Issues that must be fixed before merge.

#### CQ-{N}: {Title}
- **File**: `path/to/file.php:{line}`
- **Issue**: Description
- **Suggested Fix**: How to fix it
- **Why**: Why this matters

### WARNING
Issues that should be fixed.

#### WQ-{N}: {Title}
...

### SUGGESTION
Nice-to-haves.

#### SQ-{N}: {Title}
...

## Stats
- Files reviewed: {count}
- Critical findings: {count}
- Warnings: {count}
- Suggestions: {count}
```

Be thorough but fair. Don't nitpick formatting if it matches project conventions. Focus on real issues.
Return ONLY: "Grade: {X} — {critical} critical, {warnings} warnings, {suggestions} suggestions". Write full details to the file.
```

**Agent 2: Functionality + Plan-Diff Reviewer** (`name: "verify-functionality"`):
```
You are a functionality and plan-coverage verification agent for the Task Shredder system.

## Your Mission
Verify the implementation works correctly AND matches the plan. Write findings to: {task_dir}/verification-functionality.md

## Context
TASK: {assignment}
TASK DIR: {task_dir}

## Before You Start
Read these files in your own context:
1. `{task_dir}/plan.md` — implementation plan
2. `{task_dir}/research.md` — codebase research
3. `{task_dir}/task.json` — read the `steps.questions.qa_log` for user decisions

## What to Check

### Part A: Functionality
1. **Tests Pass**: Run `php artisan test --compact` for all new/modified test files. Report results.
2. **Test Coverage**: Are all plan items tested? Are edge cases covered?
3. **Observer Side Effects**: Do model observers fire correctly? Are there unintended cascading effects?
4. **Business Logic Correctness**: Does the code actually do what the plan says?
5. **Data Integrity**: Are database constraints correct? Foreign keys? Indexes?
6. **Edge Cases**:
   - Null values where unexpected
   - Empty collections
   - Concurrent access scenarios
   - Invalid state transitions
7. **Integration Points**: Do new features work with existing code? Check controller > action > model flow.
8. **Regression Risk**: Could these changes break existing functionality?
9. **Migration Safety**: Can migrations run without data loss? Reversible?

### Part B: Plan-Diff (MANDATORY)

Systematically verify that every planned artifact actually exists in the codebase. For EACH item in the plan's "Files Inventory" and "Implementation Steps":

1. **File existence**: For every file listed in the plan, check it exists. Use Glob/Grep.
2. **Class/method existence**: For every class and method signature in the plan, grep for it.
3. **Migration columns**: For every migration in the plan, verify the columns match.
4. **Route registration**: For every new route in the plan, run `php artisan route:list` and verify.
5. **Test existence**: For every test mentioned in the plan, verify the test file and method exist.

Produce a checklist:
```
## Plan-Diff Checklist
- [x] app/Actions/CreateRefundAction.php — EXISTS, method execute(Deal): Refund FOUND
- [x] app/Models/Refund.php — EXISTS, relationships match plan
- [ ] app/Queries/GetRefundsByDealQuery.php — MISSING (planned in Step 5)
- [x] tests/Feature/CreateRefundActionTest.php — EXISTS, 4 test methods
- [ ] Route POST /api/refunds — NOT FOUND in route:list
```

## Output Format

Write to {task_dir}/verification-functionality.md:

```markdown
# Functionality Verification: {task}

## Test Results
```
{Output of php artisan test}
```

## Summary
Overall assessment. Does it work? Grade: A/B/C/D/F

## Plan-Diff Checklist
{Checklist as described above — EVERY planned file/class/method/route/test}

## Plan-Diff Score: {implemented}/{total_planned} ({percentage}%)

## Findings

### CRITICAL
Bugs or missing functionality that must be fixed.

#### CF-{N}: {Title}
- **File**: `path/to/file.php:{line}`
- **Issue**: Description
- **Expected**: What should happen
- **Actual**: What actually happens / would happen
- **Suggested Fix**: How to fix it

### WARNING
Issues that could cause problems.

#### WF-{N}: {Title}
...

### SUGGESTION
Improvements.

#### SF-{N}: {Title}
...

## Coverage Assessment
- Plan items implemented: {X}/{total}
- Test files created: {count}
- Tests passing: {pass}/{total}

## Missing Items
List anything from the plan that wasn't implemented.
```

Run actual tests. Don't just read the code — verify it works. The plan-diff is MANDATORY, not optional.
Return ONLY: "Grade: {X} — Plan coverage: {pct}% — {critical} critical, {warnings} warnings". Write full details to the file.
```

4. When both agents complete, display a summary (from their return messages, NOT by reading the full files):
```
=== VERIFICATION RESULTS ===

Quality Review (verify-quality):
Grade: {grade}
Critical: {count} | Warnings: {count} | Suggestions: {count}

Functionality Review (verify-functionality):
Grade: {grade}  |  Tests: {pass}/{total} passing
Plan Coverage: {pct}%
Critical: {count} | Warnings: {count} | Suggestions: {count}

Total findings to address: {critical_count} critical, {warning_count} warnings
```

5. If there are ZERO critical findings AND plan coverage >= 90%, ask:
```
Clean bill of health! Skip fixes phase?
a) Yes, we're done! [Recommended]
b) No, still address the warnings/suggestions
```

   If (a): Set `steps.fixes.state = "skipped"`, `steps.verification.state = "completed"`, `status = "completed"`, `updated_at` to now. Display the final completion summary (same format as Step 8.6 but with fixes line showing `SKIPPED`). **STOP — do NOT proceed to Step 8.**
   If (b): Proceed to Step 8.

6. If there ARE critical findings or plan coverage < 90%, proceed to Step 8 directly (no skip option).

---

### Step 8: Fixes Phase — Routed Back to Implementation Agents

**Goal**: Address verification findings by sending them back to the original implementation agents via `SendMessage`. They already have full domain context, avoiding the cost of spinning up new fix agents.

1. Update `task.json`: set `steps.fixes.state = "in_progress"`, `started_at`, `current_step = "fixes"`.

2. **Do NOT read verification files yourself.** Spawn a **general-purpose sub-agent** (`name: "fix-router"`, `model: "sonnet"`) that reads both verification files and `phases-summary.json`, maps each finding to its owning phase by file path, and writes `{task_dir}/fix-assignments.json`:
   ```json
   {
     "phases_with_findings": [
       {
         "phase": 2,
         "title": "Models",
         "findings": ["CQ-1", "CF-2", "WQ-3"],
         "finding_details": "CQ-1: Missing strict types in Refund.php\nCF-2: ...\nWQ-3: ..."
       }
     ],
     "total_critical": 3,
     "total_warnings": 5,
     "total_suggestions": 2
   }
   ```
The agent returns ONLY: "{N} phases need fixes: Phase 2 (3 findings), Phase 3 (5 findings)". Read the compact `fix-assignments.json` (NOT the full verification files).

3. **Send findings back to the original implementation agents** via `SendMessage`. For each phase with findings, send the relevant findings to the corresponding `impl-phase-{N}` agent — which retains its full context from implementation, avoiding redundant research reads.

   For phases that can be fixed in parallel (no dependency conflicts), send all `SendMessage` calls simultaneously.

Display:
```
[7/7] Fixes       ======------ IN PROGRESS
Routing findings back to implementation agents — they already know the code.
  impl-phase-2 (Models): 2 critical, 1 warning
  impl-phase-3 (Logic): 1 critical, 3 warnings
  impl-phase-4 (Controllers): 0 critical, 2 warnings
```

**SendMessage template for each affected phase**:
   ```
   SendMessage(to: "impl-phase-{N}", message: "
   ## Fix Request: Verification Findings for Phase {N} — {title}

   The verification agents found issues in your domain. Fix them and commit.

   ## Findings to Fix (in priority order):

   ### CRITICAL (MUST fix):
   {Critical findings with full details from fix-assignments.json}

   ### WARNING (fix these too):
   {Warning findings}

   ### SUGGESTION (fix if straightforward, skip if complex):
   {Suggestion findings}

   ## Rules
   1. One commit per fix (or group closely related fixes)
   2. Commit format: task({slug}): fix {finding-id} - {description}
   3. Run relevant tests after each fix
   4. Do NOT touch files outside your domain

   Report: which findings fixed (by ID), which skipped and why, test results, commits made.
   ")
   ```

**Resume fallback**: If resuming the fixes step after a context reset (impl agents no longer alive), spawn fresh agents for each affected phase using the same `impl-phase-{N}` prompt from Step 6, then immediately send the fix request.

4. When all impl agents respond with their fixes, spawn a single **general-purpose agent** (`name: "fixes-logger"`, `model: "sonnet"`) to consolidate results:

**Agent prompt**:
```
Read the git log for recent task({slug}) commits and the verification files in {task_dir}/.
Write a consolidated fixes log to {task_dir}/fixes-log.md.
Run `php artisan test --compact` for a final test run and include results.
Return ONLY: "{count} findings fixed, {skipped} skipped, tests: {pass}/{total}".
```

6. Display the final summary:

```
=== TASK SHREDDER COMPLETE ===

Task: {slug}
Duration: {total_duration}

Progress:
[1/7] Research      ============ DONE
[2/7] Questions     ============ DONE ({qa_count} answered)
[3/7] Planning      ============ DONE (validated)
[4/7] Splitting     ============ DONE ({phase_count} phases)
[5/7] Implementation =========== DONE ({commit_count} commits)
[6/7] Verification  ============ DONE (Q:{grade} F:{grade} Plan:{pct}%)
[7/7] Fixes         ============ DONE ({fixes_count} applied)

Files changed: {count}
Tests written: {count}
Commits made: {count}
Findings resolved: {resolved}/{total}
Git checkpoints: {list of wave tags}

All artifacts saved in: .TaskShredder/{id}/

The task has been thoroughly shredded. Time for a coffee.
```

7. Update `task.json`: set `status = "completed"`, all steps completed, `updated_at`.

---

## Resume Logic

When resuming an existing task:

1. Read `task.json` and determine `current_step` and its `state`.

2. Display current progress with the progress bar format.

3. **Always ask the user first** before doing anything:
   ```
   This task was interrupted at step "{current_step}". What would you like to do?
     a) Continue this task [Recommended] — will restart the interrupted step
     b) Abandon this task and start a new one
     c) Roll back to a checkpoint (if implementation was in progress)
   ```
   Wait for the user's answer before proceeding.

4. **If user chose (a) — continue**: Reset the interrupted step's state to `pending`, clear any partial artifacts for that step (e.g., delete `questions-batch-*.md` if questions step), and **start the step fresh from the beginning**. All previously completed steps are preserved — only the interrupted step restarts.

   **Special case for "implementation" step**: Check which phases completed. Only re-run incomplete phases — completed phases and their commits are preserved.
   Also re-spawn the persistent `code-reviewer` agent (it does not survive context resets).

5. **If user chose (b) — abandon**: Set task status to `"abandoned"` in task.json. Then start a fresh task by asking "What task would you like to shred?"

6. **If user chose (c) — rollback**: List available git tags from `wave_tags` in task.json:
   ```
   Available checkpoints:
   1. task-shredder/{slug}/pre-wave-1 — before any implementation
   2. task-shredder/{slug}/pre-wave-2 — after Phase 1 (Database)
   3. task-shredder/{slug}/pre-wave-3 — after Phase 1, 2, 3
   ```
   After user picks a checkpoint, run `git reset --hard {tag}` (with user confirmation), update task.json to mark later phases as pending, and resume from that wave.

7. **If step state is `completed`**:
   - Advance to the next step with state `pending`.

8. **If step state is `failed`**:
   - Show the error. Offer to retry or abandon.

9. **If step state is `pending`**:
   - Start this step normally.

10. For implementation phase resume:
   - Check which phases are completed vs pending
   - Only re-spawn agents for incomplete phases
   - Completed phase commits are preserved

11. **For fixes phase resume**: The original `impl-phase-{N}` agents are no longer alive after a context reset. Re-spawn them using the same prompt from Step 6 (reading research.md, plan.md, phases.md), then send fix requests via `SendMessage`.

---

## Error Handling

- **Agent failure**: Retry up to 2 times. On each retry, include the error message in the prompt. After 2 retries, ask the user.
- **Test failure during implementation**: The implementation agent should self-fix (up to 2 attempts). If still failing, write the issue to `issues.md` and continue.
- **File ownership violation**: If a review agent detects files modified outside the phase's ownership, revert those commits and retry. If persistent, alert the user.
- **Missing files**: If research.md or other artifacts are missing when needed, re-run that phase.
- **Review rejection**: Retry implementation with feedback (up to 1 retry). If still rejected, ask the user.
- **Plan validation gaps after 2 patches**: Show remaining gaps to user and let them decide whether to proceed.
- **Issues.md blockers after 2 replannings**: Present all blockers to user for manual resolution.

---

## Progress Display Helpers

Always keep the user informed. Use these formats:

**Phase progress bar** (show after every major event):
```
=== TASK SHREDDER: {slug} ===
[1/7] Research      ============ DONE (2m 34s)
[2/7] Questions     ============ DONE (6 across 2 batches)
[3/7] Planning      ============ DONE (validated, 0 gaps)
[4/7] Splitting     ============ DONE (4 phases, 3 waves)
[5/7] Implementation ======----- IN PROGRESS (Wave 2/3)
[6/7] Verification  ------------ PENDING
[7/7] Fixes         ------------ PENDING
```

**Agent status** (when agents are working):
```
Agents at work:
  impl-phase-1: MERGED — 3 commits, review approved
  impl-phase-2: Sculpting Eloquent models... relationships are complicated.
  impl-phase-3: Writing business logic. The calculator is calculating.
  impl-phase-4: (waiting for Wave 2)
```

**Fun status messages** (rotate these, be creative):
- "Rummaging through app/Observers/ like it owes them money..."
- "Writing migrations. The database schema is getting a glow-up."
- "Reading every line like a stern code reviewer at 9am on Monday."
- "Connecting the dots between models... it's giving conspiracy board."
- "Tests are passing! The green checkmarks are chef's kiss."
- "Hunting for N+1 queries like a hawk with a profiler."
- "Vue components materializing... pixels are finding their purpose."
- "The fixer is in. Problems, meet your match."
- "Verification agents deployed. Nobody escapes their gaze."
- "File ownership review in progress — the gatekeeper is checking IDs."
- "Plan-diff scanning... every promise in the blueprint gets verified."
- "Almost there! The finish line is in sight."
