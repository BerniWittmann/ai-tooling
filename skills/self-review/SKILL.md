---
name: self-review
description: >-
  Review your own just-finished code changes with fresh eyes by spawning a
  skeptical reviewer subagent that is given only the diff. Use when you have
  finished implementing something and want to check it for correctness,
  readability, simplicity, and security before committing or moving on, or
  when the user says /self-review.
---

# Self-review

After finishing an implementation, get a clean second opinion. The point is **fresh eyes**: a subagent that did not write the code, is given **only the changes** (plus whatever surrounding code it chooses to read for context), and is told to hunt for problems rather than validate.

Do not review the changes yourself in the main context — you are biased toward the code you just wrote. Always delegate to a subagent.

## 1. Determine what changed (auto-detect)

Run in the repo root:

```bash
git status --porcelain
```

- **If there are uncommitted changes** (staged or unstaged): the review target is `git diff HEAD`.
- **If the working tree is clean**: review the current branch vs its base. Find the base (`main` or `master`, whichever exists) and target `git diff <base>...HEAD`.

If `$ARGUMENTS` names a scope (a path, a base branch, `staged`, or a PR), honor that instead.

Capture the diff and the list of changed files:

```bash
git diff HEAD            # or: git diff <base>...HEAD
git diff --stat HEAD     # changed-file summary
```

If the diff is empty, tell the user there is nothing to review and stop.

## 2. Spawn the reviewer subagent

Use the **Agent** tool (`subagent_type: general-purpose`) so the reviewer starts with clean context. Pass the **full diff** inline in the prompt. Tell it that the diff is the scope of review, but it may read the changed files and their callers for context. Use this exact prompt, with the diff appended:

```
You are a senior engineer reviewing a PR you did not write.
You are skeptical by default. Your job is to find problems, not validate.

For every changed file, check:
- Logic correctness and edge cases
- Error handling gaps
- Security issues (injection, exposure, auth)
- Unintended side effects outside the stated scope
- API contract violations
- Missing or inadequate test coverage
- Readability and needless complexity (simpler equivalent exists?)
- Consistency with surrounding conventions

You may read the changed files and their callers to confirm a finding, but
the diff below is the scope — do not review unrelated code.

Output format:
🔴 Blocking | 🟡 Should-fix | 🔵 Nitpick
One finding per line, prefixed with the emoji and `file:line`.
No praise. No summary of what the code does. If you find nothing in a
category, say nothing about it. If a finding is uncertain, say so.

--- DIFF ---
<paste the diff here>
```

## 3. Relay the findings

Report the subagent's findings to the user verbatim, grouped by severity (🔴 → 🟡 → 🔵). Do not soften or pad them. Then ask whether to fix the 🔴/🟡 items.
