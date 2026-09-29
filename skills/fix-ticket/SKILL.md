---
name: fix-ticket
description: >-
  Start a bug-fix ticket: analyze the Linear issue, reproduce it with a failing
  test, build a manual reproduction to see it broken, pause for review, then fix
  it test-first and re-run the reproduction to confirm it fixed. Use when the
  user says /fix-ticket or starts working on a bug ticket.
argument-hint: "<LINEAR-TICKET-ID>"
compatibility:
  requires:
    - mcp: linear
      description: Fetch issue details, relations, and comments
    - cli: gh
      description: Fetch linked PRs/issues (must be authenticated)
---

# Fix a Bug Ticket

End-to-end bug-fix flow for a Linear ticket: analyze → reproduce (RED) → see it
broken manually → **pause for review** → fix (GREEN) → confirm fixed.

**Input:** Linear ticket ID (e.g. `NODE-1234`, `AI-567`). If none is given, ask.

Each step below invokes an existing skill. Invoke it via the Skill tool; if a
skill is unavailable, follow its documented steps inline instead. Carry the
context forward between steps — later steps depend on earlier output.

## Steps

### 1. Analyze the ticket

Invoke the `n8n:linear-issue` skill with the ticket ID. Keep its consolidated
summary (description, comments, media, affected node, root-cause clues) as the
shared context for every step that follows.

If it stops on the `n8n-private` security gate, stop here too.

### 2. Reproduce with a failing test (RED)

Invoke the `n8n:reproduce-bug` skill, passing the full context from Step 1. Land
a failing regression test and capture its Reproduction Report and confidence.

### 3. Build a manual reproduction (BROKEN)

Before pausing, make the bug visible by hand. Save artifacts under the workspace
`.context/` dir:

- an importable workflow `.json` that triggers the bug,
- a mock-server script **only if** an external API is involved,
- a short `README` (or instructions block): how to run it, and exactly what you
  see BROKEN on the current, pre-fix code.

This exists so the bug can be seen broken first-hand at the checkpoint.

### 4. Checkpoint — STOP for review

Present, then wait for the user's go-ahead before touching source:

- root cause and the failing test,
- reproduction confidence,
- the manual reproduction and how to run it (broken).

If confidence is `UNCONFIRMED`, `SKIPPED`, or `ALREADY_FIXED`, surface that and
stop — do not proceed to a fix.

### 5. Fix test-first (GREEN)

Invoke the `tdd` skill to drive the failing test to green with the minimal
change, then refactor while green. Run tests from the affected package dir.

### 6. Confirm fixed (AFTER)

Re-run the same manual reproduction from Step 3 against the fixed code and report
the now-correct behavior — the same artifact proves broken-before / fixed-after.

### 7. Stop

No branch, no commit, no PR. When the user is ready to commit, point them at
`/n8n-commit`.
