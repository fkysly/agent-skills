---
name: codex-review-cc
description: "Battle-tested code review skill using OpenAI Codex CLI. Includes CLI pitfall avoidance, network fallback, and actionable output interpretation. Triggers: code review, review, codex review, 代码审核, 代码审查, 检查代码"
---

# Codex Code Review Skill (CC Edition)

> Battle-tested edition with CLI pitfall avoidance, network fallback strategy, and real-world case patterns.

## Trigger Conditions

Triggered when user input contains:

- "代码审核", "代码审查", "审查代码", "审核代码"
- "review", "code review", "codex review", "codex 审核"
- "帮我审核", "检查代码", "审一下", "看看代码"

## Core Concept: Intention vs Implementation

`codex review --uncommitted` only shows AI "what was done (Implementation)".
Recording intention via CHANGELOG tells AI "what you wanted to do (Intention)".

**"Code changes + intention description" → most effective AI code review input.**

---

## CLI Pitfall Guide (Must Read)

These are real pitfalls encountered in production use:

| Trap | Wrong | Correct |
|------|-------|---------|
| `-p` flag | `codex -p "review"` (`-p` = `--profile`, NOT prompt) | `codex review --uncommitted` |
| Non-interactive exec | `codex "prompt"` (opens interactive) | `codex exec --full-auto "prompt"` |
| Redundant exec | `codex exec review --uncommitted` | `codex review --uncommitted` (review is already a subcommand) |

**Key rule**: `codex review` is a direct subcommand — no need for `exec` wrapper.

---

## Network Stability & Fallback

Codex CLI can fail with `Transport error: network error` during long reviews. Once disconnected, retrying usually doesn't help.

**Fallback strategy when codex fails:**

```
1. Attempt: codex review --uncommitted (with timeout)
2. If timeout/network error → DO NOT retry endlessly
3. Fallback: manually analyze diff in current context
   - Run: git diff --stat && git diff
   - Perform review inline (check for bugs, regressions, resource leaks)
   - Report findings in same P1/P2 format
```

Always inform the user which path was taken.

---

## Execution Flow

### Step 0: Check Working Directory

```bash
git diff --name-only && git status --short
```

- **Has uncommitted changes** → Continue steps 1-4
- **Clean working directory** → `codex review --commit HEAD`

### Step 1: Ensure CHANGELOG is Updated

Check if CHANGELOG.md is in the diff:

```bash
git diff --name-only | grep -iE "changelog"
```

**If not updated, auto-generate:**

1. Analyze: `git diff --stat` + `git diff`
2. Generate entry under `## [Unreleased]` with `### Added / Changed / Fixed`
3. Write to CHANGELOG.md
4. Continue flow

### Step 2: Stage Untracked Files

Codex cannot review files not tracked by git. Stage new files:

```bash
# Check for untracked files
git status --short | grep "^??"

# Stage them safely (handles spaces/special chars)
git ls-files --others --exclude-standard -z | while IFS= read -r -d '' f; do git add -- "$f"; done
```

### Step 3: Assess Difficulty & Run Review

**Count changes:**

```bash
git diff --stat | tail -1
# e.g. "20 files changed, 342 insertions(+), 985 deletions(-)"
```

**Difficulty rules (ANY triggers xhigh):**

- Files >= 10
- Total changes (insertions + deletions) >= 500
- Insertions alone >= 300 OR deletions alone >= 300

| Difficulty | Config | Timeout |
|-----------|--------|---------|
| Hard | `model_reasoning_effort=xhigh` | 30 min |
| Normal | `model_reasoning_effort=high` | 10 min |

**Run codex review in isolated context:**

For agents that support subagent/subtask isolation (e.g. Claude Code's Agent tool), run codex in a separate context to avoid polluting the main conversation. For agents without isolation support, run directly.

**Command templates by project type:**

```bash
# Node/Bun project
bun run lint:fix 2>/dev/null; codex review --uncommitted --config model_reasoning_effort=high

# Go project
go fmt ./... && go vet ./... && codex review --uncommitted --config model_reasoning_effort=high

# Python project
ruff check --fix . && codex review --uncommitted --config model_reasoning_effort=high

# Clean working directory
codex review --commit HEAD --config model_reasoning_effort=high
```

**Timeout & failure handling:**

```
IF codex hangs or returns "Transport error: network error":
  1. Wait up to 30 seconds for recovery
  2. If no recovery → report the error message
  3. DO NOT retry more than once
  4. Trigger fallback: analyze diff manually (see Network Stability section)
```

### Step 4: Interpret Output & Act

**Priority levels:**

| Level | Meaning | Action |
|-------|---------|--------|
| **P1** | Severe bugs, regressions, data loss risk | Fix immediately |
| **P2** | Code quality, resource leaks, style | Evaluate — fix if reasonable, skip if debatable |
| **P3** | Suggestions, nits | Optional |

**Do NOT blindly follow all suggestions.** Evaluate each:

- Is this a real bug or a false positive?
- Does the suggested fix introduce new complexity?
- Is this relevant to the current change scope?

**After fixing P1/P2 items:**

1. Re-run tests to confirm fixes don't break anything
2. Optionally re-run codex for a second pass

### Step 5: Self-Correction

If codex finds CHANGELOG inconsistent with code:

- **Code is wrong** → Fix code
- **CHANGELOG is wrong** → Update CHANGELOG

---

## Real-World Case Patterns

These patterns were discovered through actual codex reviews and are worth watching for:

### Pattern: Client Navigation State Regression

**Scenario**: Using `router.push()` to navigate between pages, but component state doesn't reset because the framework reuses the component instance.

**Codex signal**: P1 — "Navigation doesn't trigger state reset"

**Fix pattern**: Use explicit state reset on navigation, or use a `key` prop to force remount.

### Pattern: Per-Request Resource Creation

**Scenario**: Creating HTTP clients/agents/connections inside request handlers instead of reusing a shared instance.

**Codex signal**: P2 — "Resource created per-request, potential leak"

**Fix pattern**: Create shared instance at module level, reuse across requests.

---

## Codex CLI Reference

### Basic Syntax

```bash
codex review [OPTIONS]
```

### Options

| Option | Description |
|--------|-------------|
| `--uncommitted` | Review all uncommitted changes (staged + unstaged + untracked) |
| `--base <BRANCH>` | Review changes relative to a base branch |
| `--commit <SHA>` | Review a specific commit |
| `--title <TITLE>` | Optional title for review summary |
| `-c, --config <key=value>` | Override config (e.g., `model_reasoning_effort=xhigh`) |

### Exec Mode (for non-review tasks)

```bash
codex exec --full-auto "your prompt here"
```

**Constraints:**

- `--uncommitted`, `--base`, `--commit` are mutually exclusive
- `[PROMPT]` is mutually exclusive with the above options
- Must run inside a git repository
