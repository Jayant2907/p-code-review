# p-code-review

> **Agent Harness** — a whole-codebase review harness for Cursor and Claude Code.

A **codebase review harness** for whole-repo audits (no diff). It orchestrates three independent review channels in parallel, aggregates them into one verdict, and writes a report.

Compatible with **Claude Code** and **Cursor**. For Claude Code, the three files in `agents/` must also be installed as custom sub-agents.

This skill **surfaces issues only** — it does not edit code. For PR / branch *diff* reviews, use a dedicated PR-review skill (`p-pr-review`) if you have one.

## What this skill does

It dispatches three isolated sub-agents on the session's selected model, then merges their JSON findings:

| Channel | Focus |
|---|---|
| **Correctness** | bugs, edge cases, test smells, dead code |
| **Security** | injection, auth, secrets, unsafe deps |
| **Reuse** | duplication, dead code, over-abstraction |

Then it:

- Marks findings **Confirmed** when two channels independently hit the same issue
- Picks the strongest verdict: `block` > `request-changes` > `approve`
- Presents a summary table to you
- Persists `reports/YYYY-MM-DD-<branch>.md` for later reference or PR attachment

Scope is the whole working tree, or paths you name. If you pass a PR number or `--branch`, stop and use a PR-review skill instead.

## When to use it

- `review this repo`
- `scan the codebase for bugs`
- `run a code audit`
- `/p-code-review` or `/p-code-review server/ scripts/`

## Install

### 1. The skill folder

#### Cursor

**User-level:**

```bash
git clone https://github.com/Jayant2907/p-code-review.git ~/.cursor/skills/p-code-review
```

Windows (PowerShell):

```powershell
git clone https://github.com/Jayant2907/p-code-review.git "$env:USERPROFILE\.cursor\skills\p-code-review"
```

**Project-level:**

```bash
git clone https://github.com/Jayant2907/p-code-review.git .cursor/skills/p-code-review
```

#### Claude Code

**User-level:**

```bash
git clone https://github.com/Jayant2907/p-code-review.git ~/.claude/skills/p-code-review
```

Windows (PowerShell):

```powershell
git clone https://github.com/Jayant2907/p-code-review.git "$env:USERPROFILE\.claude\skills\p-code-review"
```

**Project-level:**

```bash
git clone https://github.com/Jayant2907/p-code-review.git .claude/skills/p-code-review
```

### 2. Sub-agent files (required for Claude Code)

`SKILL.md` dispatches agents named `p-code-review-correctness`, `p-code-review-security`, and `p-code-review-reuse`. Copy those three markdown files into your agents directory:

```bash
# user-level
cp p-code-review/agents/*.md ~/.claude/agents/

# or project-level
mkdir -p .claude/agents
cp p-code-review/agents/*.md .claude/agents/
```

Windows (PowerShell), user-level:

```powershell
New-Item -ItemType Directory -Force -Path "$env:USERPROFILE\.claude\agents" | Out-Null
Copy-Item "$env:USERPROFILE\.claude\skills\p-code-review\agents\*.md" "$env:USERPROFILE\.claude\agents\"
```

In **Cursor**, the skill can dispatch parallel Task/sub-agents using the prompts in those files even without Claude's `~/.claude/agents/` layout; keep the `agents/` folder next to `SKILL.md` so the orchestrator can read them.

Restart the agent session so the skill is picked up.

## How to run it

- `Run p-code-review on this repo`
- `p-code-review src/ api/` (scoped paths)
- `p-code-review --sequential` (debug / tight token budget)

## What's in this repo

```
p-code-review/
├── SKILL.md                          # Orchestrator
├── README.md
└── agents/
    ├── README.md
    ├── p-code-review-correctness.md
    ├── p-code-review-security.md
    └── p-code-review-reuse.md
```

## Agent Harness collection

Part of the **Agent Harness** suite by [@Jayant2907](https://github.com/Jayant2907) — installable skills that wrap structured agent workflows (review harnesses, QA harnesses, impact sweeps) for Cursor and Claude Code.

| Harness | Role |
|---|---|
| [adversarial-review](https://github.com/Jayant2907/adversarial-review) | Merge-review harness |
| [p-code-review](https://github.com/Jayant2907/p-code-review) | Whole-codebase review harness (this repo) |
| [feature-impact-sweep](https://github.com/Jayant2907/feature-impact-sweep) | Cross-cutting impact harness |
| [j-QA](https://github.com/Jayant2907/j-QA) | Manual UI QA harness |
| [j-user-readiness](https://github.com/Jayant2907/j-user-readiness) | Visual demo-readiness harness |
| [j-module-audit](https://github.com/Jayant2907/j-module-audit) | Field-level module audit harness |
