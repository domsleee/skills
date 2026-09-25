# codex skill — eval results

Tested with subagents: each given the eval prompt, producing the `codex` command they'd run (with-skill = skill in context; baseline = no skill / model's own knowledge). Graded against the assertions in `evals.json`. Codex CLI version under test: 0.140.0, model gpt-5.5.

## Behaviour evals (evals.json)

### Iteration 1 — with-skill vs baseline

| Eval | With-skill | Baseline | Key failure in baseline |
|------|-----------|----------|-------------------------|
| 1 second opinion | 6/6 | 3/6 | `codex exec --sandbox read-only "prompt"` — no stdin redirect → **deadlocks**; no xhigh/bg |
| 2 review diff | 4/4* | 3/4 | deadlock-safe (pipe), missed xhigh |
| 3 deadlock diagnosis | 5/5 | ~3.5/5 | diagnosed deadlock but prescribed `--ask-for-approval never` — **invalid on exec** |
| 4 big doc | 4/4* | ~0.5/4 | `codex exec "$(cat design.md)…"` — no redirect → **deadlocks**; no sandbox/xhigh |
| 5 bare-codex hang | 2/2 | 2/2 | both fine; baseline added likely-invalid `--full-auto` |

With-skill ≈ 100% assertions, **0 deadlocking commands, 0 invalid flags**.
Baseline ≈ 57%, **2 commands that hang, 2 with invalid flags**.

`*` Iteration-1 flaw found in the **skill itself**: with-skill eval2 & eval4 wrote `cat file | codex exec … - <<'EOF' …instructions… EOF`. The heredoc overrides the pipe as stdin, so the file/diff is silently dropped (no deadlock, but codex reviews nothing). The `… -` example invited it.

Fix applied: documented the "instructions PLUS a file/diff" pattern — instructions in the prompt arg, content piped with **no `-`** (codex appends stdin as a `<stdin>` block), and an explicit "never combine a pipe with a heredoc" warning.

### Iteration 2 — with-skill (after fix)

All 5 = 100%. Heredoc bug gone: eval2 → built-in `review < /dev/null` + correct `git diff | codex exec "instructions"`; eval4 → `cat design.md | codex exec "instructions"` (no `-`, no heredoc, warns against it). Zero deadlocks, zero invalid flags.

Minor polish after iter2: nudged `-o FILE` over `> file 2>&1` for clean capture (eval4 used the noisy form).

### Eval 6 & 7 — real review workflow (added after reading live usage)

Reading an active session that used this skill surfaced two things:

- Eval 6 (`review --base main`) — the user's real review form; works with stdin closed.
- Eval 7 — **`codex exec review --base <branch>` cannot be combined with a custom-instructions PROMPT** (positional or piped `-`): codex 0.140.0 errors `the argument '--base <BRANCH>' cannot be used with '[PROMPT]'`. Verified directly. The skill's earlier `review --base main "instructions"` example was wrong and was fixed. Eval 7 result: with-skill PASS (pipes `git diff main...` into a plain `codex exec "…focus…"`), baseline FAIL (emitted the erroring `review --base main "…"` combo).

### Eval 8 & 9 — model defaults and astra (codex 0.154.0)

A session asked for "/codex astra high". The skill pinned gpt-5.6-sol xhigh and never named astra, so Claude ran `codex --version` and grepped `~/.codex/config.toml` to find `gpt-6-astra`. Fix: the default is now gpt-5.6-sol at `high`, and "astra" means `-m gpt-6-astra` at `medium` unless the user names an effort. Evals 1, 2, 4, 6 and 7 now expect sol at high instead of xhigh; 3 and 5 don’t check the model.

With-skill run (sonnet subagents, skill text only, no commands executed):

| Eval | Result | Command |
|------|--------|---------|
| 1 second opinion (default) | 6/6 | `cat src/auth.ts \| codex exec -m gpt-5.6-sol -c model_reasoning_effort="high" -s read-only "…"`, background |
| 8 astra, no effort | 4/4 | `cat src/cache.rs \| codex exec -m gpt-6-astra -c model_reasoning_effort="medium" -s read-only "…"`, no config lookup |
| 9 astra high | 4/4 → found a bug | `-m gpt-6-astra -c model_reasoning_effort="high"`, no config lookup, but `-i mock1 -i mock2 "PROMPT"` |

Eval 9 exposed an `-i` bug: `-i, --image <FILE>...` takes several files, so a prompt right after it is read as another image. Verified on 0.154.0: `codex exec … -i a.png "Reply with just OK" < /dev/null` prints `No prompt provided via stdin.` and exits 1; with `-- "Reply with just OK"` it answers `OK`. The earlier "exit 0" was the exit code of `tail` in the pipe, not codex. Added the `--` rule to the skill and an assertion to eval 9. Re-run: 5/5 — `-i ./mockups/*.png -- "PROMPT" < /dev/null`.

### Full run after the Codex review fixes (codex 0.154.0)

All 9 evals re-run with-skill (one sonnet subagent each, skill text only, no commands executed), graded against the current assertions.

| Eval | Result | Command shape |
|------|--------|---------------|
| 1 second opinion | 6/6 | `cat src/auth.ts \| codex exec -m gpt-5.6-sol -c model_reasoning_effort="high" -s read-only -o FILE "…"`, background, non-empty check |
| 2 review diff | 4/4 | `git diff \| codex exec … -s read-only -o FILE "Review this diff for bugs."` |
| 3 deadlock diagnosis | 5/5 | stdin-EOF deadlock, arg prompt doesn't help, `< /dev/null`, `pkill -f 'codex exec'` |
| 4 big doc | 4/4 | `cat design.md \| codex exec … "Poke holes…"`, no `-`, no heredoc |
| 5 bare-codex hang | 2/2 | `codex exec … "summarize this repo" < /dev/null` |
| 6 review vs main | 4/4 | `codex exec … -s read-only -o FILE review --base main < /dev/null` (`-o` before `review` parses — checked) |
| 7 review with focus | 4/4 | `git diff main... \| codex exec … "…focus on auth and error handling"`, no `--base`, no `< /dev/null` |
| 8 astra, no effort | 4/4 | `-m gpt-6-astra -c model_reasoning_effort="medium"`, no config lookup |
| 9 astra high | 5/5 | `-m gpt-6-astra -c model_reasoning_effort="high" -s read-only -i … -o FILE -- "…" < /dev/null` |

38/38 assertions. Zero deadlocking commands, zero invalid flags. Minor: eval 5's answer loosely called the bare-`codex` hang a stdin deadlock before correctly blaming the TUI.

## Triggering evals (trigger-evals.json)

Judged from name+description only. Expected: 1–7 trigger, 8–13 don't. The v1/v2 results below are historical 12-query runs, before query 7 (astra) was added.

- v3 description (current, with "ask codex astra"): **13/13, no mismatches** — #7 (astra) triggers, #8–13 correctly excluded.

- v1 description: 11/12. **Miss: #5** ("codex hangs forever, fix it") → did not trigger, because the description sold only "second opinion / review" and never mentioned troubleshooting hangs — even though the skill body + evals 3 & 5 are all about the stdin deadlock. Undertriggering.
- Fix: description now also advertises fixing codex calls that hang/deadlock/freeze/never-return (stdin-EOF deadlock).
- v2 description: **12/12, no mismatches** — #5 now triggers, near-misses (then #7–12, now #8–13) still correctly excluded.

## Empirical deadlock proof (Git Bash, codex 0.140.0)

- `codex exec "prompt"` (no stdin redirect) → exit 124, **hung** (killed at 60s).
- `codex exec "prompt" < /dev/null` → exit 0, returned clean.

Matches openai/codex#20919. This is the root cause of the "codex hangs every time" symptom from Claude Code's shell.
