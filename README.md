# claude-code-plugins

A curated marketplace of Claude Code plugins for workflows, coding standards, and development tools.

## Usage

### Add the marketplace

```
/plugin marketplace add ondrejehrlich/claude-code-plugins
```

### Install a plugin

```
/plugin install task-shredder@ondrej-claude-code-plugins
```

Or run `/plugin` to open the interactive plugin manager and browse available plugins.

### Recommended: Auto mode

These plugins spawn many sub-agents that read files, run commands, and write code. To avoid constant permission prompts, run Claude Code with auto-approve permissions:

```bash
claude --dangerously-skip-permissions
```

Or use [Auto Mode](https://claude.com/blog/auto-mode) for a safer alternative that still lets agents work uninterrupted while keeping guardrails in place.

> **Use at your own risk.** These modes grant Claude broad access to your filesystem and shell. Review what each plugin does before enabling auto-approve, and always work in a version-controlled repository so you can roll back if needed.

## Plugins

### Task Shredder

Shred complex tasks into parallel agent workflows with research, planning, and verification.

```
/task-shredder:shred add refund support for cancelled subscriptions
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