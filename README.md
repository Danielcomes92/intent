# intent

Two Claude Code skills that make work on a ticket precise from start to finish:

- **`/intent-spec`**: when a task arrives, turn its ticket into an agreed spec. The intended behaviour as Gherkin scenarios, the cases the ticket leaves open as questions with a proposed answer, and a definition of done.
- **`/intent-review`**: before merging, review the change against that spec, on top of Claude Code's own `/code-review`, keeping only concrete, evidenced findings.

Each project keeps what it learns in one file, `.intent/decisions.md`, so questions answered once are not asked again and decisions made on purpose are not reported as bugs.

Nothing in the plugin is specific to a project: it reads the ticket and its documents through whatever you have connected to Claude Code (Linear, Jira, GitHub or GitLab issues, Notion, Google Docs...) and the project's own rules files.

## The workflow

```
ticket arrives ──▶ /intent-spec ABC-123        spec + questions (posted to the ticket on your OK)
                        │
answers arrive ──▶ /intent-spec ABC-123 --answers   spec updated, decisions proposed for .intent/decisions.md
                        │
implement ─────▶ one test per scenario, named after it; done when the "Done when" list is green
                        │
before the PR ─▶ /intent-review ABC-123        block / review / dropped, against the spec + checks, with confidence
                        │
PR ────────────▶ paste the "Done when" checklist
```

**When to use it.** Tasks with behaviour: business rules, roles and permissions, money, dates, states, anything ambiguous. **When not.** Copy and styling changes, dependency bumps, pure refactors (the spec is "existing tests stay green"), one-line bug fixes (the spec is one scenario: the reproduction). `/intent-spec` tells you when a spec is not worth it.

**Rules of thumb.** 1–6 scenarios per task; more than ~8 means the ticket should be split. Ask only questions whose answer changes the implementation, and always propose the answer, so work continues on the clear parts while you wait. Gherkin is for agreeing on behaviour, not for tooling: write normal tests named after the scenarios, no Cucumber needed.

## Requirements

- Claude Code with its built-in `/code-review` skill.
- Optional: connectors for your tracker and docs (Linear, Jira, GitHub or GitLab issues, Notion…) so it can read tickets.
- Optional: `gh` (or your code host's CLI or connector), only to read PR details and to post reviews. Without it, the review still runs and you post the text yourself.

## Install

In Claude Code:

```
/plugin marketplace add Danielcomes92/intent
/plugin install intent@intent
```

Or from a terminal: `claude plugin marketplace add Danielcomes92/intent && claude plugin install intent@intent`. To try a local clone for one session: `claude --plugin-dir /path/to/intent`.

Update or remove:

```
/plugin marketplace update intent        # fetch the latest version
claude plugin update intent@intent
claude plugin uninstall intent@intent
```

What changed in each version: [`CHANGELOG.md`](CHANGELOG.md).

## Set up a project (once)

1. Connect the tools where your tickets and specs live (Linear, Jira, GitHub, Notion...) in Claude Code's connectors or MCP settings.
2. Optionally, tell Claude to use the workflow by adding this to the project's `CLAUDE.md`:

   ```markdown
   ## Working on tickets
   - When starting a ticket, run `/intent-spec <ticket>` before writing code; ask its questions before implementing the assumed scenarios.
   - Write one test per spec scenario, named after it.
   - Before opening a PR, run `/intent-review <ticket>` and fix or answer every **block** finding.
   - Project decisions live in `.intent/decisions.md` (only add entries the user approved).
   ```

3. `.intent/decisions.md` is created by `/intent-spec` the first time an answer is worth keeping. Commit it. Format: [`skills/intent-spec/references/decisions-format.md`](skills/intent-spec/references/decisions-format.md).
4. Optionally, add `.intent/review-checks.md` with the project's own review checks, or switch common ones off (see [Review checks](#review-checks)). Commit it too.

## Use

```
/intent-spec ABC-123                    # spec + questions for a ticket
/intent-spec ABC-123 --post             # ...and post it to the ticket after your OK
/intent-spec ABC-123 --answers          # questions answered: update spec and decisions
/intent-spec "free text describing the task"

/intent-review                          # current branch vs the default branch
/intent-review ABC-123                  # against the ticket's Intent spec
/intent-review ABC-123 --target 482     # a pull request
/intent-review --effort high             # override the effort picked from the change's size and risk
/intent-review --self                    # your own branch before a push: also rules-file deviations and comments that restate the code
```

Neither skill writes to the ticket, the PR or the repository without your approval.

## How /intent-review decides

1. **Brief**: the ticket's Intent spec (or, without one, scenarios inferred from the ticket, PR and commits, each quoting its source), out-of-scope items, and decisions made on purpose (from the spec, the rules files and `.intent/decisions.md`), and the review checks that apply to the files touched.
2. **Review**: the built-in `/code-review` runs with that brief as context.
3. **Filter**: a fresh, independent agent checks every finding against the code and keeps only concrete, evidenced defects caused by the change:
   - **block**: breaks an acceptance scenario or a project rule, or is a security, data, money/time or core-flow defect;
   - **review**: real but minor, or needs a human decision;
   - everything else is dropped and counted. Scenarios that were only *inferred* never produce a blocking finding.

4. **Reproduce**: each kept finding is run when there is a cheap way (a failing test, a script, a request, the app on a device), in a temporary worktree. It is graded **confirmed** (reproduced, says how) or **probable** (shown by the code); what doesn't hold up is dropped or asked as a question.

The full rubric is in [`skills/intent-review/references/filter-rubric.md`](skills/intent-review/references/filter-rubric.md).

### How findings are written

Each finding has a **title** (what breaks), **repro** (steps → result), **expected** (what fixed looks like and how to check it) and **why** (the cause, with `file:line`). A `probably fix:` is added only when the suggestion was applied and the repro then passed. Questions and nits are one line. Posted text is short and plain: no tables, no filler, and nothing stated as fact that wasn't run or isn't visible in the code.

When you ask it to post, it re-reads the PR first. It posts nothing on a merged or closed PR, reviews new commits before posting, and in follow-up rounds posts only approvals and new blocking findings. The full rules are in [`skills/intent-review/references/output-format.md`](skills/intent-review/references/output-format.md).

### Review checks

Every review also runs the **common checks** in [`skills/intent-review/references/common-checks.md`](skills/intent-review/references/common-checks.md). They are about 30 defects that reviews keep raising: locks released in `finally`, waits with a limit, honest error classes, invalidating what a mutation changed, optimistic rollback, no dead code or duplication, and more. The brief includes the ones that apply to the files the change touches, and the report says whether each one passes.

Change the list without editing the plugin:
- add your own checks, one per line, in `.intent/review-checks.md` (shared by the project) or `~/.claude/intent/review-checks.md` (only yours, never quoted in posted comments);
- switch a common check off with a line such as `off: ui-tokens`.

### Effort

Without `--effort`, the effort is picked from the change: **low** for small changes and re-reviews, **medium** for normal ones, **high** for payments, personal data, auth, migrations, concurrency and release branches. Risk wins over size. In our use, a higher effort mostly added candidates that the filter dropped; the evaluation below ran at `high`.

## Evaluation

Measured against plain `/code-review high` (same model, same scenarios, same ground truth) on a planted-bug benchmark of React and React Native changes: 24 tickets with acceptance criteria, 67 planted bugs (security, data fetching, hooks, React Native lifecycle, money and time, intent mismatches, architecture), decoys (correct code that looks wrong) and 4 clean changes. Half of the scenarios were used to shape the workflow; the other half was held out and only scored in aggregate.

| | `/code-review` (tuning set) | `/intent-review` (tuning set) | `/code-review` (held out) | `/intent-review` (held out) |
|---|---|---|---|---|
| Bugs found (recall) | 100% | 100% | 94% | 94% |
| Precision of all findings | 31% | **55%** | 25% | **43%** |
| Precision of blocking findings¹ | 60% | **95%** | 48% | **71%** |
| Correct code flagged as a bug (decoys) | 14 | **5** | 20 | **9** |
| Findings matching no known bug | 33 | **8** | 45 | **17** |
| Clean changes blocked | 2 of 2 | 1 of 2 | 2 of 2 | 1 of 2 |
| Cost per change (approx.) | $0.32 | $1.10 | $0.33 | $1.15 |

¹ Counting repeated reports of the same bug as correct.

The same bugs are found, with roughly half the noise and blocking findings that are far more reliable, at about three times the cost of a plain review. Known gaps: one clean change per set still gets a blocking finding, and a few real bugs land in **review** instead of **block**. The benchmark is synthetic (bugs planted by an LLM in two small fake apps), so absolute numbers are optimistic; the comparison between the two is the useful part. `/intent-spec` has not been benchmarked yet.

## Develop

```sh
git clone https://github.com/Danielcomes92/intent && cd intent
claude plugin validate .            # manifest and skills check
claude --plugin-dir .               # try your local copy in one session
```

The skills are plain Markdown: `skills/<name>/SKILL.md` holds the workflow, `skills/<name>/references/` the templates and rubrics they load. Keep them project-agnostic: no company, product or person names, no paths.

## License

MIT
