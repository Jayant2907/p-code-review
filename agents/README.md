# Required agent files

`p-code-review`'s SKILL.md dispatches three custom sub-agents by name:
`p-code-review-correctness`, `p-code-review-security`, `p-code-review-reuse`.

For the skill to work on a fresh machine, copy the three `.md` files in this
folder into `~/.claude/agents/` (user-level) or `.claude/agents/` (project-level).
Claude Code resolves custom sub-agent names from those locations — the skill
alone is not enough without them.
