---
name: codex
description: Invoke the OpenAI Codex CLI (`codex`) from inside Claude Code for a one-shot second opinion or code review, AND fix codex calls that hang or deadlock. Use whenever you shell out to `codex` — e.g. "get a codex second opinion", "ask codex", "ask codex astra", "have codex review this", "run codex on the diff", "what does codex think" — and also whenever `codex` hangs, deadlocks, freezes, or never returns when called non-interactively (the stdin-EOF deadlock). Covers the correct non-interactive `codex exec` invocation, closing stdin to prevent hangs, sandbox flags, model and reasoning effort, and capturing the answer.
---

# Invoke Codex from Claude Code

One-shot: shell out to `codex`, get its answer, done. No interactive session.

## The canonical call

```bash
codex exec -m gpt-6-sol -c model_reasoning_effort="high" -s read-only "PROMPT" < /dev/null
```

That single line is right for almost every case. The three pieces that matter:

1. **`codex exec`** — never bare `codex` (that opens the interactive TUI and hangs forever).
2. **`< /dev/null`** — the #1 cause of "codex hangs". Launched from Claude Code's shell, `codex exec` inherits an open stdin and blocks on `read()` for an EOF that never comes ([#20919](https://github.com/openai/codex/issues/20919)) — even with the prompt as an argument. A closed stdin fixes it. PowerShell can't redirect with `<`; use `cmd /c '… < NUL'` or just run it from the Bash tool.
3. **`-s read-only`** + **`-m gpt-6-sol -c model_reasoning_effort="high"`** — see below.

Run it in the **background**: it thinks for minutes.

## Feeding a file or diff

To hand Codex a large input, pipe it — put your instructions in the argument and pipe the content with **no `-`**. Codex appends piped stdin as a `<stdin>` block, and the pipe closes stdin itself (so no `< /dev/null` needed):

```bash
git diff | codex exec -m gpt-6-sol -c model_reasoning_effort="high" -s read-only "Review this diff for bugs."
cat design.md | codex exec -m gpt-6-sol -c model_reasoning_effort="high" -s read-only "Poke holes in this design (on stdin): assumptions, edge cases, security."
```

Never combine a pipe with a heredoc (`… <<'EOF'`) — the heredoc replaces the pipe as stdin and the file is silently dropped. One input channel only.

## Sandbox

Keep `-s read-only` — Codex is giving an opinion, not editing. Use `-s workspace-write` only if you actually want it to make edits; avoid `--dangerously-bypass-approvals-and-sandbox` unless the env is already isolated.

## Model + reasoning effort

| User asks for | Flags |
|---|---|
| no model (default) | `-m gpt-6-sol -c model_reasoning_effort="high"` |
| "astra" | `-m gpt-6-astra -c model_reasoning_effort="medium"` |

An effort the user names overrides the default for that model — "codex astra high" → `-m gpt-6-astra -c model_reasoning_effort="high"`. Otherwise don't drop lower to "save time"; run codex in the background and keep working.

Always pass both `-m` and the effort. `~/.codex/config.toml` defaults to gpt-6-sol at medium, so leaving them out silently runs sol at medium instead of high, and the config default can change without notice.

## Capturing the answer

Read stdout, or write the final message to a file with `-o FILE` (cleaner than `> file 2>&1`, which mixes in progress noise):

```bash
codex exec -m gpt-6-sol -c model_reasoning_effort="high" -s read-only -o answer.txt "..." < /dev/null
```

Then **check the file is non-empty** — recent builds can exit 0 with empty output when stdin is detached ([#19945](https://github.com/openai/codex/issues/19945)). Empty = retry, not "no findings". For machine-readable output: `--json` (JSONL stream) or `--output-schema FILE` (force a JSON shape).

## Code review

Built-in reviewer, scoped to a diff (still close stdin):

```bash
codex exec -m gpt-6-sol -c model_reasoning_effort="high" -s read-only review --base main    < /dev/null   # vs a base branch
codex exec -m gpt-6-sol -c model_reasoning_effort="high" -s read-only review --uncommitted  < /dev/null   # staged+unstaged+untracked
codex exec -m gpt-6-sol -c model_reasoning_effort="high" -s read-only review --commit <sha> < /dev/null   # one commit
```

**Scope flags can't be combined with custom instructions.** `--base`/`--uncommitted`/`--commit` reject a PROMPT (positional or piped `-`): `the argument '--base <BRANCH>' cannot be used with '[PROMPT]'`. So either review a scope with no instructions (above), give instructions without a scope flag (`review "focus on auth…" < /dev/null`), or pipe the diff into a plain review (`git diff main... | codex exec … "Review this diff: focus on auth."`).

## Flags (on `codex exec`)

| Flag | Why |
|------|-----|
| `-s, --sandbox <MODE>` | `read-only` (default unless config overrides — pass it explicitly), `workspace-write`, `danger-full-access`. |
| `-c key=value` | override config inline (TOML), e.g. `model_reasoning_effort`. |
| `-m, --model <MODEL>` | always pass — `gpt-6-sol`, or `gpt-6-astra` when the user asks for astra. |
| `-C, --cd <DIR>` | working root. |
| `-o, --output-last-message <FILE>` | write final answer to a file. |
| `--json` / `--output-schema <FILE>` | JSONL stream / force JSON shape. |
| `--skip-git-repo-check` | allow running outside a git repo. |
| `-i, --image <FILE>...` | attach image(s) — put `--` before the prompt (below). |

`-a/--ask-for-approval` and `--search` are **top-level-only** — on `exec` they error `unexpected argument`. `exec` needs no approval flag.

**`-i` eats the prompt.** It takes several files, so `-i a.png "PROMPT"` reads the prompt as another image, prints `No prompt provided via stdin.` and exits 1 having done nothing. End the flags with `--`: `codex exec … -i a.png -i b.png -- "PROMPT" < /dev/null`.

## When it does hang

- Kill the leftover process first (a deadlocked codex stays alive): `pkill -f 'codex exec'` (bash) / `Get-Process codex | Stop-Process -Force` (PowerShell). Then re-run with stdin closed.
- Slow ≠ hung: a working codex streams progress; a deadlocked one produces zero output and never moves. Verified on 0.140.0 — arg prompt with no redirect timed out (exit 124); with `< /dev/null`, exit 0.
- Keep prompts on the repo code; tell Codex not to wander into `~/.claude/`, skill, or agent config dirs.
