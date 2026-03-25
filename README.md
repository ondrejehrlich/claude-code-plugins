# claude-code-plugins

A curated marketplace of Claude Code plugins for workflows, coding standards, and development tools.

## Usage

### From the marketplace

```bash
claude plugin add github:ondrejehrlich/claude-code-plugins
```

### Individual plugin

```bash
claude plugin add github:ondrejehrlich/claude-code-plugins/plugins/task-shredder
```

## Plugins

### Task Shredder

Shred complex tasks into parallel agent workflows with research, planning, and verification.

```
/task-shredder add refund support for cancelled subscriptions
```

A 7-step orchestration pipeline that turns a high-level task description into fully implemented, reviewed, and verified code — without the orchestrator ever writing a line itself.

1. **Research** — 4 parallel agents scan your codebase (models, observers, business logic, frontend)
2. **Questions** — Adaptive clarifying questions presented as interactive dialogs
3. **Planning** — Detailed implementation plan with automated validation against research findings
4. **Splitting** — Phase decomposition with exclusive file ownership and dependency-ordered waves
5. **Implementation** — Parallel agents per phase with a persistent code reviewer, git checkpoints, and automatic replanning on blockers
6. **Verification** — Quality review + functionality review + plan-diff (did we actually build what we planned?)
7. **Fixes** — Findings routed back to the original agents who already have full context

All state is tracked in `.TaskShredder/` with full resume support. Interrupted tasks pick up where they left off.

See the [full documentation](plugins/task-shredder/README.md) for details.

## License

MIT — see [LICENSE](LICENSE).