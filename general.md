# general: Applies to all sessions

- Never make a technical choice without me. Assume nothing should be implicit.
- Whenever we finish a session, review it fully and update this repo with relevant informations to help the next session and for you to have less questions to ask. Use git access, not https.

## Workflow (learned 2026-11)
- Plan first, together. Ask every technical choice (design, build-system fix, script change, cancelling a job) before acting. "Your call" is the only delegation.
- When a build/test breaks: stop, diagnose to the root cause, report with evidence, propose options, wait for a decision. Don't patch around it silently.
- Long cluster jobs: use a background watcher that fails fast (job ended, unexpected jobs spawned, abnormal failure rate) instead of waiting for the end.
- Code I push is in English (identifiers, comments), even in French-named files. Translate surrounding code only if needed and only in the impacted file; ask before renaming existing API.
- If a session gets killed: the transcripts are in `~/.claude/projects/<cwd-path-with-dashes>/*.jsonl` (one dir per launch directory, e.g. `...-trust-code`). Read the tail of the latest one to resume instead of asking.
- Temporary local edits (instrumentation, old versions for a benchmark): back them up / restore with `git show HEAD:path > path`, and always restore at the end.
