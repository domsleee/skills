# claude skill — eval results

**Setup.** Real `codex exec` sessions (codex-cli 0.154.0, gpt-5.6-luna high, macOS) against scratch repos:
- `rev`: uncommitted bug in `math.js` (`avg` loop starts at `i = 1`).
- `rev7`: a `feature` branch that makes `auth.js` fail open, plus a flawed `design.md`.

Claude Code 2.1.268. Codex ran every shell command through **zsh**; bash was only covered by direct probes. Sandbox per scenario: `-s read-only` (approval `never`, so escalation is impossible) or `-s danger-full-access`. Eval 1 was run against `math.js` rather than `src/auth.ts`. Eval 8 simulates a real logout by pointing claude at an empty `CLAUDE_CONFIG_DIR`. Baseline = skill not installed; with-skill = installed to `~/.agents/skills/claude`.

## Manual probes

Observed by hand in the authoring session; raw output isn't archived here.

- Open-but-silent stdin: `claude -p` waits 3s, warns `no stdin data received in 3s, proceeding without it`, then answers. `< /dev/null` skips the wait.
- Under the Codex sandbox (`-s read-only`), claude exits 1 with `Not logged in · Please run /login`.
- An empty `CLAUDE_CONFIG_DIR`, unsandboxed, gives the **same** message and exit 1. The message alone can't distinguish sandbox from logout; only the sandbox state can.
- `claude -p --tools Read,Grep,Glob "prompt"` → `Error: Input must be provided either through stdin or as a prompt argument when using --print`. With the prompt right after `-p`, it works.
- In `-p` (no permission host), a write and a `git diff` via Bash were both denied in single haiku probes.
- Piped stdin plus a prompt argument: Claude receives both.
- Pipe plus `< /dev/null`: zsh (MULTIOS) still delivers the pipe, bash drops it. Pipe without `/dev/null` works in bash.
- `--model opus` → `claude-opus-5`. `--model fable` on this account → `You've reached your Fable limit…`, exit 1, and `--fallback-model opus` didn't rescue it. Fable was never observed answering.

## History

**Baseline (no skill), 5 runs, all failed at least one assertion:**
- Relayed a sandboxed `Not logged in` as "Claude's answer".
- Asked Claude to run `git diff` itself (it can't in `-p`).
- Passed `--tools ""` with nothing piped, so Claude reviewed blind.
- No `--model`/`--effort` on any run.
- Answered eval 3 with a generic auth checklist.

**Iteration 1 (424 words):**
- Right commands, but blamed "authentication" when escalation wasn't possible.
- Added `< /dev/null` after a pipe (only works under zsh).
- Didn't offer opus after the Fable limit.

**Iteration 2 fixes** passed all 4 scenarios, one run each.

**Iteration 3 (cut to ~210 words):**
- Read-only regressed to "not authenticated". The short wording only covered escalation being "refused".
- Diff review, Fable, and eval 3 passed.

**gpt-6-astra review #1 (skill text):**
- Took: escalate "when permitted"; cut the heading, the duplicate example, and the bash rationale.
- Rejected: its description rewrite (dropped the "run this past fable" example that fixed trigger #4), and cutting why the prompt goes first (needed for eval 5).

**gpt-6-astra review #2 (these results).** Found:
- Several PASSes not supported by the assertions (g1 counted as PASS without running unsandboxed; e3 partial).
- "Never call it an auth problem" was an overclaim, since a real logout prints the same message.
- Evals 6 and 7 hadn't been run.
- Assertions for effort and `/dev/null` were missing.
- The trigger set was tuned on the same queries it was scored on.

Response:
- Policy-conditional escalation assertions.
- New eval 8 (real logout).
- Effort and no-`/dev/null` assertions.
- Held-out trigger queries.
- This rewrite.

**Sandbox/logout wording, two more rounds after review #2:**
- *Two-branch wording* ("sandboxed → report …; allow escalation or use `-s danger-full-access`" / "unsandboxed → real logout"):
  - e8 3/3 and e3 2/2 passed.
  - e1 went **0/3**: all three blamed auth, two appended `-s danger-full-access` to the *claude* command (`error: unknown option '-s'`), and two tried `--dangerously-skip-permissions`. A Codex CLI flag in the report text read as a claude flag.
- *Final wording*: no Codex flags; "check your own sandbox mode"; report "rerun Codex with escalation allowed".

## Final version (214 words): per-assertion grades

| Eval | Runs | Grade | Notes |
|---|---|---|---|
| 1 second opinion, read-only | 5 | 4/5 | Commands correct in all 5. 4 runs reported the sandbox block. r1 reported the failure without naming a cause: no auth claim, but no sandbox attribution either. Escalation impossible here (approval `never`); none requested. |
| 2 review diff, full access | 1 | 5/5 | `git diff \| claude -p … --model opus --effort xhigh --tools Read,Grep,Glob`, no `/dev/null`, found the bug. Escalation assertion moot under full access. |
| 3 "Not logged in" diagnosis | 1 | 3/3 | Sandbox can't reach the keychain; "rerun with escalation/sandbox access allowed"; no re-login advice. (2/2 on the previous wording too.) |
| 4 Fable, full access | 1 | 4/4 | `--model fable --effort xhigh --tools Read,Grep,Glob`, ended "Fable limit reached — rerun with opus?". File reading unobserved (limit hit first). |
| 5 variadic error | 1 | 2/2 | Explained the swallow; prompt moved after `-p` (also offered stdin). |
| 6 design doc, full access | 1 | 3/3 | `cat design.md \| claude -p "Poke holes…" --model opus --effort xhigh …`; Claude found forgeable tokens, per-instance sessions, no expiry. |
| 7 branch review, full access | 1 | 4/4 | `git diff … main...HEAD \| claude -p "…focus on auth and error handling…"`; Claude flagged the fail-open catch as critical. |
| 8 real logout, full access | 2 | 1 full + 1 partial | Both said "logged out" and neither blamed the sandbox. r2 gave `claude auth login`; r1 omitted it. (3/3 on the previous wording.) |

Grades for evals 2 and 4–7 come from the version just before the final sandbox/logout wording change, which touched only that bullet.

## Triggering evals (trigger-evals.json)

Judged from name+description only by a fresh subagent. This measures classification, not real activation (in the Codex runs the skill loaded every time from eval 3's second round on).
- v1: 10/12 (missed the opus and fable model-name queries).
- v2: 12/12, 12/12, 11/12.
- Final description: 12/12 ×2 on the tuned set, then 20/20 ×2 on the expanded set, including 7/7 ×2 on **held-out** queries written after tuning.

## Known gaps

- Weakest spot: telling a sandbox block from a real logout. The message is identical, so Codex must reason from its own sandbox mode, and it occasionally reports the failure without a cause or without the fix.
- One run each for evals 2 and 4–7.
- All Codex runs used zsh.
- Fable never observed answering.
- Not covered: empty Claude output (now eval 10, unrun), a failing `git diff` hidden behind a successful pipe, and keeping Codex's own analysis separate from Claude's findings (several runs added Codex's own verdict next to Claude's).
- The nested claude loads the user's config: in eval 2 its answer ended with a claude.ai connector nag.

## Silent-run wording (2026-09-12) — not yet run

Added after three real Codex runs on Windows (codex-cli 0.154.0, gpt-6-astra high, Claude Code 2.1.269, `danger-full-access`): the reviews took 4m50s, 5m20s and 8m10s, and `claude -p` printed nothing until the end. Codex read the silence as a possible hang — `write_stdin` with `yield_time_ms: 1000` every ~5s interleaved with `sleep 45000`, roughly 10 cycles per review, narrating "still running" / "taking longer than expected" each time. All three runs succeeded and returned usable reviews; none hung.

Changes:

- Description also triggers on a `claude` call that *looks* stuck (minutes of silence), not only on the two error strings.
- Replaced the single "non-zero exit or empty output means failed" bullet with two: how to wait (one 60s `write_stdin` on the returned `session_id`; no short polls plus sleeps, no per-poll narration), and silent-but-running vs exited-and-empty.
- New eval 9 (silent run: warn up front, wait once, don't kill, don't narrate) and eval 10 (exited with no output = failed, not "no findings"), which closes the "empty Claude output" gap above.
- Three new held-out trigger queries: two positive (a silent review, "should I kill it") and one negative (an unrelated hanging build script).

**Runs (2026-09-12, Windows/PowerShell, codex-cli 0.154.0, `codex exec -m gpt-5.6-sol -c model_reasoning_effort="high"`, with-skill only):**

| Eval | Sandbox | Result | Notes |
|---|---|---|---|
| 5 variadic error | `read-only` | 2/2 | 22s. Named `--tools` as the consumer, moved the prompt after `-p`. |
| 10 exited, no output (new) | `read-only` | 4/4 | 28s. Discriminated on the new duration line: "Opus at xhigh normally takes minutes, so exiting after ~20 seconds is suspicious." Cited SKILL.md by path. |
| 3 sandbox vs logout | `read-only` | 3/3 | 24s. Still attributes the sandbox and still says not to re-login - no regression from the description/final-bullet edit. |
| 9 silent run (new) | `danger-full-access` | 6/6 | 142s total. Warned up front ("takes several minutes and prints only when it finishes... launch it once and let it complete"), one launch, two `write_stdin` waits at `yield_time_ms: 60000`, no `sleep`, no narration between launch and result. Found the planted fail-open bug and the design flaws. |

The third assertion on eval 10 (exited-and-empty vs silent-but-running) is the one the new wording exists for, and it passed by reasoning from the duration figure rather than around it.

**Still unrun:** evals 1, 2, 4, 6, 7, 8; no baseline pass for any eval, so these show correct with-skill behaviour, not lift; the expanded trigger set is unscored. Cost drove the cut - each cheap eval moved the weekly codex limit about one point, with ~12% left before a 2026-09-15 reset, and evals 2/6/7 are live opus reviews at eval 9 cost.

Two corrections from these runs:

- The claude call in eval 9 took 98s, not the 5-8 minutes seen on the real PRs - rev7 is three small files, so duration tracks diff size. SKILL.md now says 1.5-8 min (n=4) and that it scales with the diff. Eval 10 still discriminates correctly, because a 20s exit is short against either end of that range.
- `yield_time_ms: 60000` does hold in codex-cli 0.154.0: the wait returned `wall_time_seconds: 60.0065`. The eval 9 assertion is testing real behaviour, not an unsupported flag.

Platform caveat: the graded history above is macOS/zsh; these runs are Windows/PowerShell with a different model (sol vs luna), so they are not directly comparable to the earlier numbers. The 5-8 minute figure remains n=3 on one machine - an expectation, not a bound, and not evidence that a live process can never hang.

Reviewed by gpt-6-astra (medium) before the edit. Taken: "poll sparingly" was too vague to change the observed 45s loop; empty output needed splitting into running vs exited; the wait mechanism had to be concrete; the 5–10 minute claim was unsupported. Rejected: its proposal to rewrite the sandbox/logout bullet so it no longer attributes a cause — it was reviewing SKILL.md without these results, that bullet is the eval 3 / eval 8 pair that took two rounds to settle, and its wording is close to iteration 3's, which regressed to "not authenticated".


## Streaming update (2026-09-14)

The grades above describe earlier skill versions. Eval 9 now expects streaming
and checks final completion; evals 11–13 cover empty thinking text, a missing final
result, and duplicate answer text. These revised/new scenarios are **ungraded**.

Manual probes on macOS, Claude Code 2.1.270, Opus xhigh:

- The three streaming flags emitted initialization after 1.4s and thinking-token
  estimates after 4.6s. There were 43 thinking-progress events; thinking-text
  deltas were empty. A nonempty successful result arrived at 86.4s; exit 0 at 87.4s.
- The first review of this PR also streamed progress and returned a successful
  result after 162.4s. This is a live CLI check, not an independent Codex eval.
- Opus flagged possible loss of exit status/errors through filters and loss of
  output through truncation. The revised skill retains raw NDJSON/stderr logs,
  checks Claude's own status, and extracts the final result from the full log.

Frontmatter and JSON validation pass. Previous duration figures are observations,
not bounds or a test for whether a process has failed.
