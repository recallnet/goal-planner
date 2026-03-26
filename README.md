# goal-planner

Claude Code plugin for planning and filing goals and feature issues through conversation.

## What it does

`/goal-planner` guides you through creating well-formed GitHub issues:

- **Goals** — high-level intent with success criteria. No specs. Label: `goal`.
- **Features** — commit-sized sub-issues with specs, acceptance criteria, and TDD requirements. Label: `feature`. Linked as sub-issues of their goal.
- **Multi-goal ordering** — breaks large work into ordered goals with `blocked_by` relationships.
- **Ready gate** — nothing gets labeled `ready` until you explicitly confirm.

## Install

```bash
claude plugin install recallnet/goal-planner
```

Then use `/goal-planner` in any Claude Code session.

## Factory-aware routing

If the repo has a `factory.config.json` with `target_repo` and `control_plane_repo`, the plugin routes issues to the right place:

- Target repo work → files on `target_repo`
- Factory changes → files on `control_plane_repo` with `factory:<id>` label

In a plain repo (no factory config), everything files on the current repo.

## npm package

Published as `@recallnet/claude-goal-planner` on npmjs for version tracking and distribution.

## License

MIT
