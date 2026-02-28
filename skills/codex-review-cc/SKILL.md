---
name: codex-review-cc
description: "Run code review using OpenAI Codex CLI to get a second-opinion from a different AI model. Use this skill whenever the user asks for code review, codex review, 代码审核, 代码审查, 检查代码, review, 审一下, or 看看代码. This skill delegates the actual review to the `codex` CLI tool — do NOT review the code yourself."
---

# Codex Code Review Skill

This skill delegates code review to the **Codex CLI** (`codex review`), which is powered by OpenAI's models. The whole point is to get a **second opinion from a different AI** — if you (Claude) review the code yourself, the user gets zero additional value. That's why the codex CLI call is the core action, not a nice-to-have.

## Execution Flow

### 1. Pre-flight checks

Run these in parallel to assess the situation:

```bash
git diff --name-only && git status --short
```

```bash
git diff --stat | tail -1
```

- If the working directory is clean (no changes), use `codex review --commit HEAD` instead of `--uncommitted`
- If there are untracked files (`??` in status), stage them first — codex can't see untracked files:
  ```bash
  git ls-files --others --exclude-standard -z | while IFS= read -r -d '' f; do git add -- "$f"; done
  ```

### 2. Ensure CHANGELOG exists in the diff

Check: `git diff --name-only | grep -iE "changelog"`

If CHANGELOG is missing from the diff, generate one before running codex. Codex reviews better when it can compare "what was intended" (CHANGELOG) vs "what was done" (diff):

1. Read `git diff --stat` and `git diff` to understand the changes
2. Write an entry under `## [Unreleased]` with `### Added / Changed / Fixed`
3. Save to CHANGELOG.md

### 3. Run codex review (the core action)

Pick the effort level based on diff size:

| Condition (any one triggers "hard") | Effort |
|-------------------------------------|--------|
| Files >= 10, or total lines changed >= 500, or insertions >= 300 or deletions >= 300 | `model_reasoning_effort=xhigh` |
| Otherwise | `model_reasoning_effort=high` |

Run the codex CLI via Bash tool. Use a 10-minute timeout for normal reviews, 30-minute for hard ones:

```bash
codex review --uncommitted --config model_reasoning_effort=high
```

For clean working directories:
```bash
codex review --commit HEAD --config model_reasoning_effort=high
```

Optionally prepend a lint-fix step for the project's ecosystem (e.g. `bun run lint:fix 2>/dev/null;` for Node/Bun projects).

**CLI pitfalls to avoid:**
- `codex review` is a direct subcommand — do NOT wrap it in `codex exec`
- `-p` means `--profile`, NOT prompt — never use `codex -p "review"`
- The command runs non-interactively and prints its review to stdout

### 4. Interpret codex output and act

Read codex's review output and categorize findings:

| Level | Meaning | Action |
|-------|---------|--------|
| **P1** | Bugs, regressions, data loss risk | Fix immediately |
| **P2** | Code quality, resource leaks | Fix if reasonable, explain if skipping |
| **P3** | Nits, style suggestions | Optional |

Evaluate each finding critically — codex can produce false positives. Check:
- Is this a real bug or a misunderstanding of the codebase?
- Does the suggested fix introduce new complexity?
- Is it relevant to the scope of this change?

After fixing P1/P2 items, run tests to confirm nothing broke.

### 5. If codex fails: fallback to manual review

Codex CLI sometimes fails with `Transport error: network error` or hangs. If the codex command fails or times out:

1. Do NOT retry more than once
2. Tell the user codex failed and you're falling back to manual review
3. Read the diff yourself (`git diff --stat && git diff`) and perform the review inline
4. Report findings in the same P1/P2/P3 format

The user should always know which path was taken.

## Patterns worth watching for

These recurring issues have been caught by codex in real reviews:

- **Navigation state regression**: `router.push()` reuses component instances, state doesn't reset. Fix: explicit reset or `key` prop.
- **Per-request resource creation**: HTTP clients/agents created inside handlers instead of shared. Fix: module-level singleton.
