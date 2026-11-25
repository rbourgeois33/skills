# general: Applies to all sessions

- Never make a technical choice without me. Assume nothing should be implicit.
- Whenever we finish a session, review it fully and update this repo with relevant informations to help the next session and for you to have less questions to ask. Use git access, not https.

## Workflow (learned 2026-11)
- Plan first, together. Ask every technical choice (design, build-system fix, script change, cancelling a job) before acting. "Your call" is the only delegation.
- When a build/test breaks: stop, diagnose to the root cause, report with evidence, propose options, wait for a decision. Don't patch around it silently.
- Prefer minimal, scoped fixes (e.g. guarded by the condition that causes the problem) over broad changes.
- Code style for ports: look like the surrounding code, readable for a beginner over performance, few comments, one code path (no CPU fallback), remove dead code/error machinery when asked.
- Don't edit sources while a queued job will still compile them (e.g. a baseline build): draft in a scratch file, apply after the binary is built.
- Long cluster jobs: use a background watcher that fails fast (job ended, unexpected jobs spawned, abnormal failure rate) instead of waiting for the end.
