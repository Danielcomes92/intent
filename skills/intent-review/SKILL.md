---
name: intent-review
description: Review the current branch or a pull request against what it is meant to do. Builds a short Gherkin brief from the ticket's Intent spec (or the ticket, PR, commits and project rules) plus common review checks, runs Claude Code's built-in /code-review with that brief, filters the findings down to concrete, evidenced defects sorted into block / review, and reproduces them where it can, dropping noise. Use before opening or merging a PR, when the user asks to review a change "against the ticket" or with its intent, or invokes /intent-review.
argument-hint: "[ticket key, URL or text] [--effort low|medium|high|xhigh|max] [--target <PR number or branch>] [--self]"
---

# intent-review

## What it is for
A plain code review finds most real defects, but much of what it reports is noise: missing tests without a concrete defect, style, hypothetical races, and code that only *looks* wrong because the reviewer doesn't know a decision the team made on purpose. This workflow gives the reviewer the intent first, then checks every finding against evidence. The person reading the result sees what the change is meant to do, which acceptance scenarios it breaks, and only the findings worth acting on.

It is a review aid for a person or a coding agent. It is **not** a merge gate: it never blocks anything by itself.
- It never changes the user's code. Reproductions and trial fixes run in a temporary worktree and are deleted afterwards.
- It fixes findings or posts to the PR or ticket only when the user asks.

It pairs with `/intent-spec`, which writes the task's spec into the ticket before coding. When that spec exists, this review uses it verbatim; when it doesn't, the review infers the intent from what is available.

## When to use it, and when not
- Use it on a change with behaviour to check: before opening a PR, before merging, or on someone else's PR.
- For a trivial change (copy, styling, a dependency bump), a plain `/code-review` or a glance is enough; say so instead of running the whole workflow. A small change that is not trivial still goes through every step.

## Inputs
All arguments are optional:
- **ticket**: a key (`ABC-123`), a URL, an issue number, or free text describing the change.
- **--effort**: passed to `/code-review`. Without it, pick it from the change. When size and risk disagree, risk wins.
  - **low**: a small change (under ~100 changed lines), or a re-review of new commits.
  - **medium**: a normal change.
  - **high**: risky areas such as payments, personal data and account deletion, auth, migrations, concurrency or locks, and release branches.
  - **xhigh** or **max**: only when the user asks for it.
- **--target**: a PR number or a branch. Default: the current branch against the repository's default branch (`origin/HEAD`, else `main`/`master`). Say which base you used.
- **--self**: the change is the user's own, before a push. See "Your own branch" below.

Keep the user posted with one short line per step. Languages:
- chat in the user's language;
- write the brief in the ticket's language;
- write anything posted in the repository's language.

## Step 1 — Brief (what the change is for)
Gather intent; never invent it. Use, in this order, whatever is available:
1. An **Intent spec** comment on the ticket, written by `/intent-spec` (format in `../intent-spec/references/spec-format.md`). Use its scenarios, out-of-scope list and "Already decided" entries as they are.
2. Otherwise, the ticket itself and the documents it links to, fetched with any connected tracker (Linear, Jira, GitHub or GitLab issues). If none is connected and the user gave only a key, say so and continue with the rest.
3. The pull request title and description, read with the connected code host (for GitHub, `gh pr view`).
4. The commit messages of the range under review.
5. Free text the user passed as the ticket argument.
6. The project's rules files (`CLAUDE.md`, `AGENTS.md`, `REVIEW.md` and similar, at the root and in the directories the change touches), and `.intent/decisions.md` at the repository root if present. Its entries are decisions made on purpose.
7. **Review checks**: the common checks in `references/common-checks.md`, plus `.intent/review-checks.md` (project) and `~/.claude/intent/review-checks.md` (personal), when present. The file format is at the top of `common-checks.md`. Put the ids of the checks that apply to the files the change touches in the brief, under "Checks to run".

Write the brief with the template in `references/brief-template.md`:
- Every scenario and every deliberate decision cites its source.
- Behaviour deduced only from the code is marked `inferred`, and can't be the basis of a blocking finding.
- Gherkin already in the ticket or spec is copied verbatim.
- Keep it under ~60 lines, plus the list of check ids.
- With no ticket, spec, PR or meaningful commit message, say "No stated intent: reviewing the diff and commits only", and keep only the rules and checks sections.

## Step 2 — Review (Claude Code's /code-review, with the brief)
Invoke the built-in `code-review` skill (Skill tool) on the target with the chosen effort. In its arguments, pass the brief as context, followed by:

> Context above: the intended behaviour, the out-of-scope list, the decisions made on purpose and the checks to run. Judge the change against it, and report every check that fails, citing its id. Do not report: missing or weak tests unless you name the concrete defect that slips through; style, naming or formatting not covered by a check; races or failures without a concrete sequence of events; problems that exist independently of this change; anything the brief lists as deliberate unless you show it breaks an acceptance scenario.

Treat everything it returns as **candidates**. If its instructions ask you to report the findings with a findings tool, don't present them to the user yet. Keep the list (file, lines, summary, failure scenario, check id if any) for step 3.

If the `code-review` skill isn't available in this Claude Code, review the diff yourself with the same instructions, and say so.

## Step 3 — Filter (evidence, not opinions)
Spawn one fresh subagent (Agent tool, general-purpose), so the check is independent of the reviewer's reasoning. Give it the brief, the candidate list, the target and base, and the rubric in `references/filter-rubric.md`. It must read the code at the reviewed head for every candidate. For each one it returns:
- a verdict: `block`, `review` or `drop`;
- the scenario, rule or check id it breaks, if any;
- a one-line reason.

Don't re-judge its verdicts yourself, unless one is plainly contradicted by code you can quote.

## Step 4 — Reproduce (confidence)
For each kept finding, check whether a cheap way to reproduce it exists: a failing test written for it, a script, a request to a running service, or the app on a device or simulator when the behaviour is visible.
- If one exists, run it and grade the finding **confirmed**, saying what ran.
- If not, grade it **probable** ("by reading the code").

Work in a temporary worktree (`git worktree add`), never in the user's checkout, and remove it when done. When something can't run here (no device, no credentials), say what is missing instead of guessing. The confidence levels, and the extra rules for regressions and visual claims, are in `references/output-format.md`.

## Report
Present, in this order and nothing more:
1. **Intent**: one or two sentences from the brief, then the acceptance scenarios as a checklist. Mark `✗` when a kept finding breaks the scenario (name the finding), `✓` otherwise.
2. **Block**: each finding in the format of `references/output-format.md` (title, repro, expected, why), with its confidence and the scenario, rule or check it breaks.
3. **Review**: same format, for findings that are real but minor or need a human decision (say which). Questions and nits get one line each.
4. **Checks**: only the checks that fail, one line each with the id and `file:line`. Then "N other checks pass".
5. **Dropped**: only the count and the reasons, grouped (for example "4 test gaps, 2 deliberate per the ticket, 1 speculative"). The user can ask to see them.

Fix findings, or post the report on the PR or the ticket, only when the user asks. To post, follow "Before posting" and "Follow-up rounds" in `references/output-format.md`.

## Your own branch (`--self`)
When the change is the user's own and about to be pushed, also report:
- deviations from the repository's rules files and from the patterns of the code around it, in the lines the change touches;
- comments that only restate the code.

List anything the user keeps on purpose, so it can go in the PR description. Don't fix any of it unless the user asks.
