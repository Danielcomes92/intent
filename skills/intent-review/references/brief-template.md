# Brief template

If the ticket has an **Intent spec** comment (from `/intent-spec`), use its scenarios, out-of-scope list and "Already decided" entries as they are, and only add the rules and checks sections. Otherwise fill this from the ticket, PR, commits, rules files and `.intent/decisions.md`. Quote sources; do not invent.

```markdown
## Intent
<1–2 sentences: what the change must achieve, for whom.>
Sources: <ticket key / PR #n / commits>

## Acceptance scenarios
Scenario: <name>                     [explicit | decided | assumed | rule | inferred]
  Given <state>
  When <action>
  Then <observable result>
  Source: "<quote>" (<ticket / PR / CLAUDE.md:line / file for inferred>)

<one scenario per acceptance criterion, then the edges the criteria imply:
 empty data, another role or tenant, offline, time zone, retries — only when relevant>

## Out of scope
- <what the ticket or PR says this change does not do> (source)

## Deliberate decisions
- <behaviour that may look wrong but is intended> — "<quote>" (source)

## Rules that apply
- "<quoted rule>" (<rules file:line>)

## Checks to run
- <id> (common | project | personal)
```

Labels: `explicit`, `decided` and `assumed` come from an Intent spec, where `assumed` depends on an open question. `rule` comes from a rules file, and `inferred` is deduced from the code only. Only `explicit`, `decided` and `rule` scenarios can back a blocking finding.

Example of a good scenario (explicit, quoted):

```gherkin
Scenario: a declined payment never shows the success screen   [explicit]
  Given a customer whose card is declined
  When they confirm the payment
  Then they see the error message and stay on the payment step
  Source: "AC2: a failed payment must never land on the confirmation page" (SHOP-7)
```
