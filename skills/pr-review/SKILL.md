---
name: pr-review
description: >-
  Review a GitHub PR assigned to you for review (someone else's PR) by running
  two reviewers in parallel — the generic skeptical self-review reviewer and,
  for n8n PRs, the n8n:human-like-code-review skill — then aggregate both into a
  per-issue plan: severity, how to fix, and a ready-to-paste review comment in
  your own voice.
  READ-ONLY: never posts, replies, or resolves anything on GitHub; you do that.
  Use when given a PR URL to review (an incoming/assigned PR), or when the user
  says /pr-review. For addressing review comments on YOUR OWN PR, use
  /pr-review-feedback instead.
---

# PR review

Review an incoming PR (one assigned to you, that you did **not** write) and hand
back a per-issue action plan: what's wrong, how severe, how to fix it, and a
comment you can paste yourself. Two reviewers run in parallel and their findings
are merged. You handle every GitHub interaction — this skill only reads and advises.

## Hard rule: read-only

**Never** post comments, replies, reviews, reactions, resolve threads, approve,
request changes, or any other write to GitHub. Use only read commands
(`gh pr view`, `gh pr diff`, `gh api` GETs). Pass this constraint to every
subagent. If a write would help, write the text for the user to paste — do not
do it. This is the whole point of the skill.

## 1. Get the PR

The PR URL (or `owner/repo#number`) comes from `$ARGUMENTS` or the user's
message. **If none was given, ask for it and stop** — do not guess.

Fetch the PR once, read-only:

```bash
gh pr view <pr-url> --json title,body,state,headRefName,baseRefName,author,authorAssociation,reviews,comments,url
gh pr diff <pr-url>                                            # the diff = review scope
gh api "repos/{owner}/{repo}/pulls/{number}/comments" --paginate   # existing inline comments
```

Note existing review comments so the reviewers don't repeat points already raised
(if a point is resolved, confirm it's handled; if still open, you may reinforce it).

Determine if this is an **n8n PR**: the URL owner is the `n8n-io` org (or another
n8n repo), and the `n8n:human-like-code-review` skill is available. This decides
whether the second reviewer runs.

Note `authorAssociation` too: `FIRST_TIME_CONTRIBUTOR` / `CONTRIBUTOR` /
`NONE` means an outside contributor, which changes the register (see Voice below).

## 2. Launch the two reviewers in parallel

Spawn both with the **Agent** tool (`subagent_type: general-purpose`) in a
**single message** so they run concurrently. Tell each: read-only, the diff is
the scope, return findings as text (no GitHub writes).

**Reviewer A — generic skeptical review (always).** This is the
[`self-review`](../self-review/SKILL.md) reviewer applied to this PR. Read that
file and pass its §2 reviewer prompt verbatim, with the PR diff appended. (If
`self-review` is missing, fall back to: "You are a senior engineer reviewing a PR
you did not write. Skeptical by default — find problems, don't validate. Check
each changed file for logic/edge-case bugs, error handling, security, unintended
side effects, API-contract breaks, missing tests, needless complexity, and
comment quality. Output `🔴 Blocking | 🟡 Should-fix | 🔵 Nitpick`, one finding
per line prefixed with the emoji and `file:line`. No praise, no summary.")

**Reviewer B — n8n human-like review (only for n8n PRs where the skill is
available).** Run a subagent from an n8n repo working directory and tell it to
invoke `/n8n:human-like-code-review` for `<pr-url>` (fallback: read that skill's
SKILL.md and follow it). Have it return the full findings text **and** the path
to the markdown file it writes. This reviewer adds the n8n lenses the generic one
lacks: backward compatibility / node versioning, established-convention checks,
and human, copy-paste-friendly comment phrasing.

If it's not an n8n PR (or the skill is unavailable), skip Reviewer B and say so —
only the generic review ran.

## 3. Aggregate into a per-issue plan

Merge both reviewers' findings. **Dedup** overlapping points (same file:line /
same concern) into one entry, crediting both sources. Group by file, order by
severity (blocking first).

Emit each issue in exactly this shape. The triage fields are for the user to
read; the suggested comment is for the user to **copy**, so it goes in a fenced
block of its own, flush to column 0, with **no leading indentation on any line**
— indentation is what makes a paste unusable.

`````
### <file>:<line> — 🔴/🟡/🔵
**Raised by:** generic · human-like · both
**What's wrong:** <one-line restatement, grounded in the actual code>
**Needs addressing?** Yes / Optional / No — <why>
**How to fix:** <concrete change — exact symbols/lines, a snippet if it clarifies>

**Suggested comment:**

````markdown
<the comment, written in the user's voice — see Voice below>
````
`````

Rules for that block:

- Use a **four-backtick** fence tagged `markdown`, so a nested ```` ```suggestion ````
  or code fence inside the comment survives intact.
- Nothing inside the fence may be indented — no list continuation indents, no
  hanging indents under `**What's wrong:**`, no wrapping the block in a bullet.
  The fence content is the literal comment body, byte for byte.
- Never put the comment inline after a `**label:**`. It always gets its own fence.
- One fence per issue. If a finding needs both a prose comment and a separate
  `suggestion` patch, put both in the same fence, as the user would type them.

Calibrate honestly — don't rubber-stamp. **Yes** = real bug / blocking / clear
win. **Optional** = judgment call (give the tradeoff). **No** = the finding is
mistaken, out of scope, or a misread (say why, and what the user might reply).
If a reviewer is wrong, say so.

## Voice — how suggested comments must read

Written from 116 of the user's own review comments on other people's PRs
(n8n-io repos, May–Aug 2026). The goal is a comment they can paste without
editing, because it already sounds like them.

**Shape.** Short: median 181 characters, 77% under 300. One point per comment.
Plain prose — no bold labels, no headings, no bullet scaffolding for a single
point. Bullets only when there genuinely are 2+ separate questions ("Two
questions on this:"). Backticks for symbols, paths and values. A fenced block
only to show a real payload or shape; a table only for old-vs-new behaviour.
`suggestion` blocks are rare (1 in 116) — use one only when the exact
replacement text *is* the point, e.g. a copy fix.

**Register.** Casual, lowercase-leaning, unpolished. 52% of their comments start
with a lowercase word ("should we add some tests?", "i think this method here is
affected as well"); 19% use a bare lowercase "i"; contractions often lose the
apostrophe ("dont", "isnt", "wont", "thats"). Match that register — but don't
manufacture typos. Shorthand they actually use: `sth`, `aka`, `e.g.`, `TBH`.

**Their openers**, in rough order of frequency: "I think…" (22%), "Maybe we
should…" / "maybe" (14%), "this…", "should we…?", "thinking about…", "I'm
wondering about…", "curious about…", "not sure whether…", "i would expect…",
"For me the question is:", "What about…?", "Why…?", "Does…?".

**Moves that make it theirs:**

- Lead with the observation, then the consequence. Not "this is a bug" but "this
  results in the following shape: … so for me when testing it still wires it to
  the success branch".
- Ground it in something checked: what they tested, the payload they saw, a link
  to upstream docs or library source (9% of comments carry a link), a before/after
  table. "When testing this against a mock server, `gid` was not adequately
  stripped: …" beats "this may not handle default values".
- Reason forward to user impact, often as a hypothetical: "I'm imagining the
  following future situation: …", "might unexpectedly change behavior for users
  when the api changes", "i think this code would break top level filters for
  existing workflows".
- Attach the alternative. "So i think this line needs to be `…`", "we should wrap
  each in try catch or use sth like `Promise.allSettled`", "how about extracting
  the parser once (e.g. `credentials/common/google-scopes.ts`) — keeps the two
  from drifting again".
- Ask when it's genuinely a question — 40% of their comments contain one, and a
  question is their default for anything they haven't verified.
- Downgrade small stuff out loud, not just with an emoji: `nit:` / `nitpick:`, or
  in words — "probably a nitpick", "Dont think this needs adressing, but wanting
  to just raise this so we think about it", "might be a non-issue/nit".
- Leave room to be wrong: "Or am i missing sth here?", "not sure about that
  myself", "(maybe not having enough context to judge this)". Use this whenever
  the finding rests on a reviewer's inference rather than something you checked.
- Missing tests get a blunt standing ask: "do we have tests for this?", "should we
  add some tests? i dont see any tests in the diff?", "please add tests to ensure
  that this previous logic is covered in the new code as well". "please add tests"
  is about as imperative as they get.
- If the finding came from tooling and can't be fully stood behind, say so —
  they do: "a finding from my reviewing skill: …", "Ai raised that one, i dont
  fully understand it myself, so just posting it here: …".
- Emoji only for praise or thanks (🙌 🚀 `:+1:`), never inside a criticism, and
  sparingly (5 of 116).

**Never** (absent from all 116, and it would read as not-them): "Consider…", "It
would be advisable…", "Please note that…", "I would recommend that you…", "Great
work, but…", praise sandwiches on inline comments, severity theatre in the
comment text ("this is critical", "blocking issue") — severity lives in the plan,
not in what gets pasted. No ticket IDs or plan/phase references inside suggested
code comments; they actively flag those in review.

**Outside contributors** (`authorAssociation` is not `MEMBER`/`OWNER`) get a
warmer top-level register, and only there: open with thanks ("First of all, thank
you very much for your effort and this contribution", "Thanks for your
contribution 🙌"), then the asks as a short numbered list, then a close. Inline
comments stay in the normal voice.

## 4. Summarize

End with a short triage list: blocking issues, quick wins, deferrable/declinable
— so the user can act fast. If Reviewer B wrote a markdown file, link it last as
`[<path>](<path>)` so it opens directly.

Do not make code changes unless the user asks for that as a separate step. The
default deliverable is the plan, not edits — and never a GitHub write.
