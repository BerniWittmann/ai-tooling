---
name: trim-comments
description: >-
  Aggressively delete low-value comments — the ones that restate what the code
  already says, narrate steps, banner sections, or log the diff ("now uses X").
  Deletion is the default; shortening is the exception. Strips issue references
  (NODE-123, #456) from every comment it keeps. Keeps real why-comments,
  directives, and load-bearing API docs. Use when the user says /trim-comments,
  asks to remove/reduce/clean up unnecessary or AI-generated comments, or says
  the code is over-commented.
argument-hint: "[path|glob] [--report] [--all]"
---

# Trim comments

Comments earn their place by saying something the code cannot. Everything else
is debt: it goes stale, it lies after the next edit, and it pushes the real code
off the screen. AI-written code over-comments by default — this skill removes
that noise without touching behaviour.

**Two rules that override everything else:**

1. **Only comments change. Not one line of code.**
2. **Delete beats shorten.** A shortened comment that still says nothing is not
   a win — it is the same debt, reformatted. Cut the whole block unless a
   specific, non-obvious fact inside it survives on its own.

## 1. Scope

| Arg | Scope |
| --- | --- |
| *(none)* | The current change: `git diff HEAD`, or `git diff <base>...HEAD` if the tree is clean |
| a path / glob / dir | Those files, in full |
| `--all` | Whole file(s) of every file touched by the current change, not just added lines |

Default scope is **added lines only** — a comment that was already on master is
someone else's decision (surgical-changes rule). `--all` opts out of that.

```bash
git status --porcelain
git diff HEAD                    # or: git diff origin/master...HEAD
```

Empty scope → say so and stop.

## 2. Hard rule: no issue references, ever

No comment may reference a ticket or issue. Not `NODE-1234`, not `#33892`, not
`ENG-77`, not a `linear.app/...` or `github.com/n8n-io/n8n/issues/...` URL, not
`Defense #2` or a phase/step number from a design doc. This holds for **every**
comment — including ones you otherwise keep, and including `TODO` / `FIXME`,
`describe`/`it` names, and commit-style annotations.

Why: ticket IDs rot faster than the code. Tickets get renamed, split, archived;
the reference outlives its meaning and sends the next reader on a dead hunt.
That context belongs in the PR description and the ticket itself.

How to apply:

- The comment is *only* the reference (`// NODE-1234`, `// see #456`) → delete it.
- The reference wraps a real fact → strip the reference, keep the fact.
  `// NODE-4988: redact before turndown escapes underscores`
  → `// turndown escapes underscores, so redact before it runs`
- `// TODO(NODE-1234): drop once v2 lands` → `// TODO: drop once v2 lands`.
- **Links to upstream sources are not issue references and stay** — vendor API
  docs, RFCs, library source permalinks, upstream bug reports in *someone
  else's* tracker. Those explain why the code is shaped this way and don't rot
  the same way. `// Google drops payloads with both fields set: <docs url>` stays.

Report every stripped reference — this rule fires even in `--report` mode.

## 3. Cut these — delete the whole comment

Default action for everything in this list is **delete**, not shorten:

- **Restatement** — the comment is the code read aloud.
  `// increment the counter` above `counter++`; `// loop over items`;
  `/** Returns the user id. */` on `getUserId()`.
- **Step narration** — `// Step 1:` … `// 2. Then we validate` walking through a
  function that already reads top-to-bottom.
- **Section banners** — `// ---- Helpers ----`, `// === Types ===`, unless the
  file uses them consistently as an existing convention.
- **Diff narration** — comments written *for the reviewer of this change*, not
  the next reader: `// now uses the new API`, `// added for the retry case`,
  `// previously this was inline`, `// simplified`.
- **Type echo** — JSDoc `@param {string} name - the name` where the signature is
  already typed, and the prose adds nothing. If *every* line of a doc block is
  inferable from the signature, delete the block — don't leave a one-line stub.
- **Restated-obvious guards** — `// null check`, `// early return`, `// cleanup`,
  `// defensive check`, `// sanity check`.
- **Vague gestures** — `// for performance`, `// for safety`, `// needed for
  compatibility`, `// handle edge cases`, `// important`. These name a category
  without naming the fact. Unless you can replace it with the *specific* reason
  from the code, it goes. A gesture at a reason is not a reason.
- **Preambles and signposts** — `// Helper to …` on a function named `formatX`,
  `// Note that …` followed by something already obvious, `// Here we …`.
- **Commented-out code** — delete it; git has it. Flag it separately in the
  report so it's an obvious call rather than a silent one.
- **Anything carrying an issue reference** — per §2.

## 4. Keep these — never cut

- The **why**: a specific non-obvious invariant, ordering constraint, workaround,
  or tradeoff. It names a fact, not a category.
  `// turndown escapes underscores, so redact before it runs`.
- **Directives** — `eslint-disable*`, `@ts-expect-error`, `@ts-ignore`,
  `biome-ignore`, `prettier-ignore`, `istanbul ignore`, `c8 ignore`,
  `v8 ignore`, `@vitest-environment`, `#region`, coverage/bundler pragmas.
- **`TODO` / `FIXME` / `HACK` / `XXX`** — keep the marker and its text, strip any
  issue reference (§2).
- **Load-bearing API docs** on exported symbols — the parts a caller cannot read
  off the signature: thrown errors, units, valid ranges, mutation/side effects,
  deprecations, non-obvious defaults, "must be called after X". Delete the
  padding around them. A doc block that only re-describes the name is not
  load-bearing, even on a public export.
- **License / copyright / generated-file headers.**
- **Comments naming a specific security, perf, or compatibility fact** — the ones
  that explain why a safe-looking simpler version is wrong. Note the difference
  from a vague gesture above: `// for security` goes, `// user input reaches the
  shell here, so quote before interpolating` stays.
- **Links to upstream docs, specs, or library source** that justify the code.
- Anything inside a **string, template literal, regex, fixture, or snapshot**
  that merely looks like a comment.

## 5. The keep test — you must justify keeping, not cutting

Strict default: **cut unless you can name the save.** For every comment you keep,
you must be able to finish this sentence in a few concrete words:

> Without this line, the next reader would ___.

If the blank needs "…understand it slightly less quickly", cut it. If it needs
"…re-derive that Notion rejects mixed-type formula branches" or "…reorder these
two calls and break redaction", keep it. Vagueness in the justification is
evidence the comment is vague.

Genuinely 50/50 after that test → **cut it**, and list it in the report so the
call is visible and reversible with one `git checkout`. Do not use "borderline"
as a place to park comments you didn't want to decide on; git is the safety net,
not the keep list.

## 6. Rewrite only when a fact survives

Rewriting is the **exception**, not the middle path. Rewrite only when a bad
comment contains a specific fact that is worth keeping and is not in the code.
Then keep the fact and drop everything else — one line, present tense, about the
code beside it.

```diff
-// Loop through the items and, because the upstream API sometimes returns
-// duplicates (see NODE-1234), we keep a Set of seen ids here so we
-// don't process the same one twice. Step 3 of the pipeline.
+// upstream can repeat ids within one page
```

If you cannot state the surviving fact in one clause, there is no surviving fact
— delete the whole comment instead. Never shorten a restatement into a shorter
restatement.

Also delete a comment that is now **wrong**. A stale comment is worse than a
noisy one, and "fixing" it is only worth it if the corrected version would pass
the keep test on its own. If the code and the comment disagree, the code wins;
say so in the report.

**Self-check:** deletions should clearly outnumber rewrites. If you rewrote more
than you deleted, you were too soft — go back through the rewrites and apply §5
to each one.

## 7. Verify

After editing, prove nothing but comments moved:

```bash
git diff -U0 -- <touched files>
```

Read it. Every `-` line must be a comment line (or a blank line orphaned by a
removed block). Any code line in the diff is a bug in your own edit — restore it
immediately.

Then, if the change touches a package with fast checks, run its lint/typecheck to
catch a directive you removed by accident. Report the actual result; don't assume.

## 8. Report

```
Trimmed comments — <scope>

Deleted   N   (restatement 6, narration 3, vague gesture 2, diff-narration 1)
Rewrote   N   ← should be well below Deleted
Stripped issue refs  N
Deleted commented-out code  N   ← call these out individually
Kept-but-close-call  N

  path/to/file.ts:42   deleted    restatement — "// increment the counter"
  path/to/file.ts:88   rewritten  narration → kept the dedup fact
  path/to/other.ts:12  stripped   "NODE-1234: " prefix removed, fact kept
  path/to/third.ts:57  cut (50/50) — gestured at perf without naming it
```

For anything kept as a close call, give the one-line justification from §5 so the
user can overrule it. `--report` stops here: list the findings, change nothing.
Without it, apply the edits and print the same report as a record of what changed.
