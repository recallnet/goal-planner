---
name: goal-planner
description: Plan and file goals and feature issues through conversation. Use when the user wants to create goals, plan features, file issues, or discuss what to build next. Handles goal decomposition into features, multi-goal ordering, and ready-label confirmation.
---

# Goal Planner

You help users plan work and file it as well-formed GitHub issues for execution.

## Setup

Determine the target repo. Check in order:

1. If a `factory.config.json` exists (or `factories/*/factory.config.json`), read `target_repo` and `control_plane_repo` from it.
2. Otherwise, use the current repo from `gh repo view --json nameWithOwner --jq .nameWithOwner`.

If a factory config exists with a `control_plane_repo` different from `target_repo`, the user may want to file factory-change issues on the control plane. Ask if the work is for the target codebase or the factory itself.

## Conversation Flow

### 1. Understand what the user wants

Listen. Ask clarifying questions. Understand the problem before proposing a solution. Read relevant code if needed to understand feasibility and scope.

### 2. Draft the goal issue

A goal is a high-level intent. It describes *what* and *why*, never *how*.

**Goal format:**
```markdown
## Goal Statement

{1-2 paragraphs: what we want to achieve and why it matters.}

## Success Criteria

- [ ] {Observable outcome 1}
- [ ] {Observable outcome 2}
- [ ] {Observable outcome 3}

## Scope

**In:** {What's included}
**Out:** {What's explicitly excluded}
```

**Goal rules:**
- No implementation specs. Ever. The goal says what, not how.
- No code snippets, file paths, or technical approach in the goal body.
- Success criteria are observable outcomes, not implementation tasks.
- Label: `goal`. Do NOT add `ready` yet.

Present the draft to the user. Iterate until they approve.

### 3. Optionally decompose into feature issues

If the conversation makes it clear what features need to be built, offer to create feature sub-issues. Only do this if the user agrees — some goals are better left for later decomposition.

**Feature format:**
```markdown
## Parent Goal
#<goal-number>

## What
{Clear description of this specific feature. What it does, where it fits.}

## Spec
{Technical specification. File paths, function signatures, contracts, wiring.
Be specific — this is what a coding agent will implement.}

## Acceptance
- [ ] {Testable criterion 1}
- [ ] {Testable criterion 2}

## TDD
{What failing test to write first.}
```

**Feature rules:**
- Each feature should be roughly one commit of work.
- Features DO contain specs, acceptance criteria, and TDD requirements.
- Label: `feature`. Do NOT add `ready` yet.
- All features in a goal will be built together on one worktree, then merged as a set.
- Set parent relationship:
  ```bash
  child_id=$(gh api repos/<repo>/issues/<feature_number> --jq '.id')
  gh api -X POST repos/<repo>/issues/<goal_number>/sub_issues -F sub_issue_id=$child_id
  ```

### 4. Handle multi-goal ordering

If the work is too large for one goal, break it into multiple goals. Use GitHub issue dependencies to enforce ordering:

```bash
# Make goal 2 blocked by goal 1
goal1_id=$(gh api repos/<repo>/issues/<goal1_number> --jq '.node_id')
goal2_id=$(gh api repos/<repo>/issues/<goal2_number> --jq '.node_id')
gh api graphql -f query="mutation { addIssueDependency(input: {issueId: \"$goal2_id\", dependsOnIssueId: \"$goal1_id\"}) { issue { id } } }"
```

Tell the user: "Goal 1 will fully complete before Goal 2 starts."

### 5. Confirm and label ready

After all issues are created, present the full plan:

```
Here's what I've created:

Goal: #<number> — <title>
  Feature: #<number> — <title>
  Feature: #<number> — <title>
  Feature: #<number> — <title>

Nothing will build until you confirm. Say "ready" to label everything
and start execution, or tell me what to change.
```

**Only label `ready` after explicit user confirmation.** The user must say "ready", "go", "ship it", "looks good, do it", or something unambiguously affirmative.

When confirmed:
```bash
gh issue edit <number> --repo <repo> --add-label "ready"
```

Apply to the goal and all its features at once.

### 6. Check for duplicates first

Before creating any goal, check existing open goals:
```bash
gh issue list --repo <repo> --label goal --state open --json number,title --jq '.[] | "#\(.number) \(.title)"'
```

## What NOT to do

- Do not label `ready` without explicit user confirmation.
- Do not put implementation details in goal issues.
- Do not create features without user agreement to decompose.
- Do not guess the target repo — detect it from config or git remote.
- Do not create circular blocked-by relationships.
- Do not file duplicate goals.

## Factory-aware routing

When a factory config exists with `control_plane_repo` different from `target_repo`, and the user describes a factory problem (bad prompts, missing capabilities, broken gates):

- File on `control_plane_repo` instead of `target_repo`
- Add label `factory:<factory_id>` alongside `goal`
- The goal statement should describe the factory behavior change

This routing only activates when the factory config is present. In a plain repo, everything files on the current repo.
