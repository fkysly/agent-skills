---
name: codex-review-cc
description: "Run code review using OpenAI Codex CLI to get a second-opinion from a different AI model. Use this skill whenever the user asks for code review, codex review, 代码审核, 代码审查, 检查代码, review, 审一下, or 看看代码. This skill delegates the actual review to the `codex` CLI tool — do NOT review the code yourself."
---

# Codex Code Review Skill

This skill delegates code review to the **Codex CLI**, which is powered by OpenAI's models. The whole point is to get a **second opinion from a different AI** — if you (Claude) review the code yourself, the user gets zero additional value. That's why the codex CLI call is the core action, not a nice-to-have.

## Execution Flow

### 1. Pre-flight checks

Run these in parallel to assess the situation:

```bash
git diff --name-only && git status --short
```

```bash
git diff --stat | tail -1
```

- If there are untracked files (`??` in status), stage them first — codex can't see untracked files:
  ```bash
  git ls-files --others --exclude-standard -z | while IFS= read -r -d '' f; do git add -- "$f"; done
  ```

### 2. Choose review mode

After pre-flight checks, use AskUserQuestion to let the user pick the review mode. Present the diff summary (file count, lines changed) so they have context:

```
AskUserQuestion:
  question: "检测到 {N} 个文件变更，{insertions}+ {deletions}-。选择审查模式："
  header: "Review mode"
  options:
    - label: "通用审查"
      description: "审查所有变更，找 bug、回归、代码质量问题 (codex review --uncommitted)"
    - label: "定向审查"
      description: "针对特定角度审查，比如测试质量、安全性、性能 (codex exec)"
```

- User picks **通用审查** → Go to **Step 3a**
- User picks **定向审查** → ask a follow-up question for specifics if the user's original request doesn't already contain a clear focus, then go to **Step 3b**
- User picks **Other** and types a custom response → interpret their response and route accordingly

If the user's original request already clearly specifies a focus (e.g. "review 测试代码找冗余"), you can skip the question and go directly to **Step 3b**.

### 3a. Generic review (codex review)

For broad "just review everything" requests. Ensure CHANGELOG exists in the diff first (`git diff --name-only | grep -iE "changelog"`), generating one if missing — codex reviews better when it can compare intent vs implementation.

Pick the effort level based on diff size:

| Condition (any one triggers xhigh) | Effort |
|------------------------------------|--------|
| Files >= 10, total lines >= 500, insertions >= 300, or deletions >= 300 | `model_reasoning_effort=xhigh` |
| Otherwise | `model_reasoning_effort=high` |

```bash
codex review --uncommitted --config model_reasoning_effort=high
```

For clean working directories: `codex review --commit HEAD --config ...`

Use 10-minute timeout for normal, 30-minute for xhigh.

### 3b. Focused review (codex exec)

For requests with a specific review angle. Compose a prompt and run it through `codex exec --full-auto`.

**Compose the prompt** from three parts:

1. **Objective** — what the user wants reviewed, in their words
2. **Scope** — which files to focus on (derive from user's request; e.g. "test code" → `*.test.ts` files)
3. **Output format** — always ask for P1/P2/P3 structured findings

**Prompt template:**

```
Review the codebase with this specific focus:

OBJECTIVE: {user's concern, e.g. "Find redundant tests, fake tests that don't actually verify anything meaningful, and tests that would pass even if the implementation was broken"}

SCOPE: {file pattern, e.g. "All test files (*.test.ts, *.test.tsx)"}

INSTRUCTIONS:
- Read the relevant source files to understand what the tests SHOULD be verifying
- Then read each test file and evaluate whether each test case actually tests something real
- Consider: does this test add confidence? Would it catch a real regression? Or is it just going through the motions?

Report findings as:
- P1 (Critical): Tests that are actively misleading (appear to test something but don't)
- P2 (Important): Redundant tests, tests that overlap significantly, low-value assertions
- P3 (Minor): Style improvements, better test organization

For each finding: file path, test name, what's wrong, and suggested fix.
```

Adapt the template to fit the user's actual concern — the example above is for test quality review. For a security review, the instructions and priority definitions would be different.

**Run it:**

```bash
codex exec --full-auto "YOUR_COMPOSED_PROMPT" --config model_reasoning_effort=high
```

Same effort/timeout rules as 3a. Use a 10-minute timeout for normal, 30-minute for xhigh.

**CLI pitfalls:**
- `codex review` is a subcommand for generic diff review — do NOT wrap it in `exec`
- `codex exec --full-auto "prompt"` is for custom prompts — this is the focused mode
- `-p` means `--profile`, NOT prompt — never use `codex -p`

### 4. Interpret codex output and act

Read codex's output and categorize findings:

| Level | Meaning | Action |
|-------|---------|--------|
| **P1** | Bugs, misleading code, data loss risk | Fix immediately |
| **P2** | Quality issues, redundancy, resource leaks | Fix if reasonable, explain if skipping |
| **P3** | Nits, style | Optional |

Evaluate each finding critically — codex can produce false positives. Check:
- Is this a real issue or a misunderstanding of the codebase?
- Does the suggested fix introduce new complexity?
- Is it relevant to the scope of the review?

After fixing P1/P2 items, run tests to confirm nothing broke.

### 5. If codex fails: fallback to manual review

Codex CLI sometimes fails with `Transport error: network error` or hangs. If the codex command fails or times out:

1. Do NOT retry more than once
2. Tell the user codex failed and you're falling back to manual review
3. Read the relevant files yourself and perform the review inline
4. Report findings in the same P1/P2/P3 format

The user should always know which path was taken.
