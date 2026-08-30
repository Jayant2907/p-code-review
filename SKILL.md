---
name: p-code-review
description: Use when the user asks to review or audit the current codebase without any diff — e.g. "review this repo", "scan the codebase for bugs", "run a code audit". For PR / branch diff reviews, use p-pr-review instead. Surfaces issues only — does not edit code.
---

# Code Review — Codebase Audit

Orchestrate three independent review channels in **parallel**, each
running as a dedicated sub-agent on the session's selected model, then
aggregate their structured findings into a single verdict, present the
summary to the user, and persist the report to
`reports/YYYY-MM-DD-<branch>.md` so it's available for later reference
or PR attachment.

Scope: **whole-codebase audit** — no diff. Each channel scans the
entire working tree (or the paths the user names) with its own lens.
For diff-based PR / branch reviews, use the sibling `p-pr-review`
skill.

## Why sub-agents

Each channel covers a different lens (correctness / security / reuse)
and they don't share state. Running them sequentially in the main
context bloats the controller's window. Sub-agents fix that:

- **Isolated contexts.** Each channel runs in its own context window,
  so the controller stays small and free to do the merge work.
- **Session model inheritance.** Channels don't pin a model — each
  inherits whatever model the user selected for the session, so the
  user's `/model` choice governs cost and depth uniformly.
- **Parallel wall-clock.** Dispatching all three in one message
  collapses total time from `sum(3 channels)` to `max(3 channels)`.

## Resolve the review target

| Invocation | Target |
|---|---|
| `/p-code-review` | Whole working tree at `.` |
| `/p-code-review server/ scripts/` | Scoped scan across the named paths |
| `/p-code-review --sequential` | Same target, channels run sequentially (debug only) |

State the resolved target clearly before dispatching. Resolve the
`roots` to the current working directory when the user passes no paths,
or to the paths they explicitly name. If the user passes what looks
like a PR number, a branch name, or `--branch`, stop and tell them to
use `p-pr-review` instead — that's a different skill.

## Phase 1 — Dispatch three sub-agents in parallel

**Send ONE message containing three `Agent` tool uses** so Claude Code
runs them in parallel. Three sequential messages defeats the purpose.

| Sub-agent | Channel | Focus |
|---|---|---|
| `p-code-review-correctness` | Correctness / quality | bugs, edge cases, test smells, dead code |
| `p-code-review-security` | Security / data safety | injection, auth, secrets, unsafe deps |
| `p-code-review-reuse` | Reuse / simplification | duplication, dead code, over-abstraction |

All three inherit the session's selected model — do not pass a model
override when dispatching.

Each dispatch prompt body:

```
roots: <resolved paths>
focus: <one-line description of what the codebase does, if known>
budget: <soft wall-clock ceiling, e.g. "3 min">
```

Each sub-agent returns ONLY a JSON object. Schema:

```json
{
  "channel": "correctness" | "security" | "reuse",
  "verdict": "approve" | "request-changes" | "block",
  "findings": [
    { "severity": "critical|high|medium|low",
      "file": "<path or ''>",
      "line": <integer or null>,
      "summary": "<headline>",
      "details": "<rationale>" }
  ]
}
```

### `--sequential` escape hatch

When the user passes `--sequential` (debugging an individual channel,
or working under a hard token budget that can't afford 3 parallel
contexts), dispatch the agents one at a time instead. Same agents,
same inputs, same merge — just serialised.

## Phase 2 — Parse and validate

For each of the three responses:

1. Attempt to parse as JSON.
2. If parsing fails, fall back to free-form display for that channel
   only: keep the agent's raw output in the final summary under a
   "Channel X — raw output (JSON parse failed)" section. Do NOT
   abandon the whole review.
3. If parsing succeeds, validate the shape: `channel` matches the
   dispatched name, `verdict` is one of the allowed values, `findings`
   is an array, each finding has the required fields. Drop malformed
   findings; keep the rest.

## Phase 3 — Merge findings across channels

A finding from one channel is **Confirmed** when another channel
reports a finding referring to the same issue. Match heuristics, in
order:

1. **Exact match on (file, line)** when both are non-null.
2. **Same file, line within ±5** when both are non-null.
3. **Same file, summary substrings overlap** (first 40 chars
   case-insensitive) when one or both lines are null.

Group matched findings together; mark them `Confirmed (channels: A, B)`.
Singleton findings are left as `(channel: X)`.

Pick the **overall verdict** as the strongest across channels:
`block > request-changes > approve`. If any channel blocks, the merged
verdict is block.

## Phase 4 — Present to user

Render this summary verbatim:

```markdown
## Code Review Summary

**Target:** <paths>
**Channels run:** Correctness · Security · Reuse

### Verdict
<one of: ✅ Approve / ⚠️ Request changes / 🚫 Block>

<one-paragraph overall assessment — synthesise the three channels'
verdicts, call out any "Confirmed by 2+" critical findings prominently>

### Findings

| Severity | Finding | Channel(s) | File:Line |
|---|---|---|---|
| Critical | <summary> | Security, Correctness | path/to/file.go:42 |
| High | <summary> | Correctness | path/to/other.go:128 |
| Medium | <summary> | Reuse | — |
| Low | <summary> | Correctness | — |

Sort the table by severity (critical → low), then by "Confirmed by 2+"
status (multi-channel findings before singletons within a severity).

### Required before merge

- <bulleted blocking items, one per critical+confirmed finding>
- <or "None" if verdict is Approve>

### Suggested follow-ups

- <non-blocking items the user should look at>
- <or "None">

**Report saved to:** reports/YYYY-MM-DD-<branch>.md
```

## Phase 5 — Save the report

After presenting to the user, persist the same Markdown you just
rendered to a stable path so it can be attached to a PR, forwarded to
a teammate, or diffed against a later run.

### Resolve the path

1. Get today's date in `YYYY-MM-DD`. Prefer the shell's local date
   (`date +%Y-%m-%d`); fall back to UTC if unavailable.
2. Get the current branch name via `git rev-parse --abbrev-ref HEAD`.
   Sanitise: replace `/` with `-`, strip anything that isn't
   `[A-Za-z0-9._-]`, trim to 80 chars. If the working tree is not a
   git repo, use `nogit` in place of the branch.
3. Construct: `reports/YYYY-MM-DD-<branch>.md`.
4. If that path already exists (a second run on the same day/branch),
   suffix with `-1`, `-2`, … to avoid clobbering the earlier report.

### Ensure the folder is ignored by git

Before writing, check `.gitignore` for an entry that already covers
`reports/`. Coverage rules kept conservative (literal-only, plus
obvious dir-covers-child):

- **Exact match**: user has `reports/` or `reports` — skip.
- **Ancestor**: user has a parent that covers it (rare; unlikely to be
  intentional at this level) — skip.

If no covering entry exists, append `reports/` on its own line to the
end of `.gitignore` (create the file if missing). Do NOT wrap it in a
sentinel block — this is a plain user-visible edit and shouldn't be
managed by any tool sentinel.

### Write the report

Use `Write` to create `reports/YYYY-MM-DD-<branch>.md` with the exact
Markdown from Phase 4 above (including the header, verdict, findings
table, required-before-merge list, and suggested-follow-ups list). Add
a small preamble at the top:

```markdown
# Code Review Report

**Generated:** <ISO-8601 timestamp with timezone>
**Repository:** <basename of the working directory>
**Target:** <same as Phase 4>

---
```

After writing, print one line to the user: `Report saved to
reports/YYYY-MM-DD-<branch>.md`.

## Red Flags

| Thought | Reality |
|---|---|
| "Channel 1 already caught everything; skip the others" | Run all three. Each has a different lens — security misses are the most expensive. |
| "Dispatch the agents one at a time so I can read the output as it arrives" | That's `--sequential` — explicit. Default is parallel; collapsing to sequential by accident wastes wall-clock. |
| "The agent returned prose, not JSON. I'll re-dispatch." | Don't. Fall back to free-form display for that channel only. Re-dispatching costs more than degrading one channel's presentation. |
| "All three findings sound similar; just keep one" | The merge logic flags Confirmed findings explicitly — that's the signal. Don't collapse silently; the user wants to see which channels agreed. |
| "Security finding is marginal; downgrade to medium so it doesn't block" | Don't downgrade. The security channel is the dedicated adversarial lens — it exists to call shots the others can't. If it says critical, treat as critical. |
| "The user just wanted the summary in chat — skip the file write" | Phase 5 is not optional. Users depend on the file being written; a later PR-attachment step assumes it exists. Print + write both. |
| "reports/ isn't in .gitignore; the review file is fine to commit though" | It's not. Reports may contain excerpts of production secrets from the security scan, and they mutate every run. Add `reports/` to .gitignore. |
| "The user gave me a PR number; run it through this skill anyway" | No. Route them to `p-pr-review` — this skill has no diff channels. |

## What this skill does NOT do

- **No PR / branch diff review.** If the user wants a diff-based review
  of a PR, a branch, or the current branch vs main, use `p-pr-review`.
- **No automatic fixing.** Findings are surfaced to the user. Fixes are
  the user's call (often delegated to `p-fix-defect` or
  `p-fix-ado-pr-comments` afterward).
- **No commenting on the PR.** Posting comments to GitHub / ADO is
  outside this skill's scope. The user copies the summary themselves
  (or points the PR bot at the saved report file) if they want it on
  the PR.
- **No re-dispatching on failure.** A sub-agent that returns garbage
  degrades to free-form; we don't retry. Retries inflate token cost
  without solving the underlying issue.
- **No cross-run diffing.** Successive report files sit side by side
  in `reports/`. Comparing them is up to the user or a follow-up
  skill.
