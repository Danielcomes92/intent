# Changelog

## 0.4.0

`/intent-review`:
- **Reproduce step.** Each kept finding is run when there is a cheap way (test, script, request, device), in a temporary worktree, and graded **confirmed** or **probable**.
- **Common review checks** (`references/common-checks.md`): about 30 recurring defects (locks released in `finally`, waits with a limit, honest error classes, invalidating what a mutation changed, optimistic rollback, no dead code or duplication…). Add your own or switch any off in `.intent/review-checks.md` (project) or `~/.claude/intent/review-checks.md` (personal).
- **How findings are written and posted** (`references/output-format.md`): title / repro / expected / why, `probably fix:` only when tested; short posted text; re-read the PR state right before posting; follow-up rounds post only approvals and new blocking findings.
- **Effort picked from the change** (low / medium / high, risk over size) instead of a fixed `high`.
- **`--self`** for your own branch before a push: also reports rules-file deviations and comments that restate the code.
- Brief labels aligned with `/intent-spec` (`explicit`, `decided`, `assumed`, `rule`, `inferred`).

Plugin: requirements and update/uninstall instructions in the README, `homepage` and `repository` in `plugin.json`.

## 0.3.2

`/intent-spec`: investigate first, decide last, never decide what others own.
