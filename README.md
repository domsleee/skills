# skills

Two skills that let Claude Code and Codex ask each other for a second opinion.

## `codex` for Claude Code

Lets Claude Code ask Codex to review code, plans or docs.

```shell
bunx skills add domsleee/skills/skills/claude-only -g -a claude-code
```

Then ask Claude Code things like:

- "Get /codex astra medium to review it."
- "Loop with /codex astra high until it passes review."
- "Plan it out, then get /codex astra max to review the plan."

## `claude` for Codex

Lets Codex ask Claude (Opus by default) to review its work.

```shell
bunx skills add domsleee/skills/skills/codex-only -g -a codex
```

Then ask Codex things like:

- "Use /claude to get Opus to review it."
- "When you're done, use the claude skill to review it with Opus xhigh."
