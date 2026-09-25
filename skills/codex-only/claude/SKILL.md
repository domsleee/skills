---
name: claude
description: Use when the user wants a second opinion, review, or answer from Claude, named directly or by model ("ask opus", "run this past fable"), or when a `claude` CLI call fails with "Not logged in · Please run /login" or "Input must be provided either through stdin or as a prompt argument", or looks stuck — minutes of silence with no output.
---

Capture streaming output in temporary files (Bash example; adapt temp-file and redirection syntax for other shells):

```bash
claude_log=$(mktemp)
echo "$claude_log"
claude -p "PROMPT" --model claude-opus-5-5 --effort xhigh --tools Read,Grep,Glob \
  --output-format stream-json --verbose --include-partial-messages \
  < /dev/null > "$claude_log" 2> "$claude_log.err"
```

- Run with `sandbox_permissions: "require_escalated"` when permitted. If claude prints `Not logged in · Please run /login`, check your own sandbox mode:
  - `read-only` or `workspace-write`, not escalated: the sandbox blocked it. Report "claude can't run inside the Codex sandbox; rerun Codex with escalation allowed". It is not an auth problem and not Claude's answer.
  - `danger-full-access` or escalated: a real logout. Tell the user to run `claude auth login`.
- Keep the prompt right after `-p` (`--tools` swallows a later prompt).
- `--model fable` only if the user asks. On `You've reached your Fable limit`, end with "Fable limit reached — rerun with Opus 5.5?"
- These tools can't run commands, so pipe diffs and docs on stdin (`git diff | claude -p "Review this diff." …`) and omit `< /dev/null` when piping.
- Record the printed log path. Between session waits, run a separate short command to read complete NDJSON lines from that file (ignore an unfinished last line) and print only the latest progress: `system/thinking_tokens` (`estimated_tokens`, when present), tool activity, or `system/api_retry`. The redirected session itself stays silent. Thinking text can be empty; filtering only answer text hides progress. Retain the full log so output truncation cannot lose the final answer.
- Inspect stderr and error events for the login/Fable messages above. If using a live pipe instead of file redirection, keep stderr separate, flush filter output (`jq --unbuffered`, not `--slurp`), and preserve Claude's exit status; the last filter's status is not sufficient.
- Opus 5.5 at xhigh can run for minutes; mention this when launching. Launch once; reuse the execution session with `write_stdin(session_id, chars: "", yield_time_ms: 60000)`, not short polls plus sleeps. Do useful work meanwhile; don't narrate every empty poll. Silence alone does not justify killing, relaunching, or changing model, effort, or MCP settings.
- After exit, extract the final `result` from the full log; don't concatenate it with partial text or complete assistant messages. Require Claude's exit code 0, `subtype: "success"`, `is_error: false`, and a nonempty answer. Missing results, errors, or empty answers mean the review failed, not "no findings"; diagnose login/Fable errors using the rules above.
