# Output and posting

How kept findings are written for the user, and the rules for anything that goes on a PR or ticket.

## Confidence

Every kept finding carries one, and says how it was reached:

- **confirmed**: reproduced by running something that can be run again: a failing test, a script, a request, an app run on a device or simulator. Say what ran and what it showed.
- **probable**: the code shows it directly, but nothing was run. Say "by reading the code".
- **possible**: a suspicion the filter or the reproduction did not hold up. It is not a finding: leave it out, or ask it as a one-line question.

Only when you call something a **regression**: run it several times on both the reviewed head and its merge base, in the same environment. Behaviour that also fails on the base is not caused by the change.

**Visual** claims (layout, clipping, colours, icons) need the real render on the target platform. A proxy such as an image tool or a mock is not enough.

## A finding

Use four short blocks, in this order:

1. **title**: what the user sees or what breaks, in one bold line.
2. **repro**: numbered steps, then `→` and the result. When it was run several times, add the count (for example `→ 3/3`).
3. **expected**: what "fixed" looks like and how to check it, so the author can verify the fix alone. For example: `expected: the total shows 2 decimals. to check: add 3 items at 0.10, the total is 0.30`.
4. **why**: the cause in one or two sentences, with `file:line`.

There is no `fix:` block. Write `probably fix:` only when the suggestion was applied and the same repro then passed. Even then it says "probably", because the author knows the code better. An untested suggestion is left out.

**Questions and nits** are one line each. A nit with no impact is dropped.

## Wording of anything posted

- Use the repository's language, short and plain. No emojis, no tables, no closing filler ("worth a quick look", "let me know").
- Only state as fact what the code shows directly or what you ran yourself and can run again. Everything else is phrased as a possibility or a question.
- Comment on code only. Product or scope decisions taken by the ticket owner are not debated in review comments; mention them to the user instead.
- When the flagged code came from another PR merged into this branch, or arrived after an earlier review, say so.
- When a repro needed a review-only patch (a mock or a forced flag), name the exact change so the author can repeat it.
- For a visual finding, attach the screenshot or video. If you can't upload it, give the user the file path to attach.

## Approvals and request-changes

- **An approval** is one short line or an empty body. Don't recap what was tested.
- **A request-changes review with several points** gets a one-line body saying which point blocks.

## Before posting (only when the user asks to post)

Right before the post, in the same step, re-read the PR state with the connected code host (for GitHub: `gh pr view <n> --json state,isDraft,headRefOid,reviews`). Stop when:

- the PR is **merged or closed**: post nothing, and tell the user. A finding worth keeping goes to a ticket.
- the **head moved** since the review: review the new commits first, then show the user what changed.
- **someone else already approved** and you have nothing that blocks: ask the user before adding anything.

Without a way to read the PR state, show the text and let the user post it.

## Follow-up rounds

Post only what changes something for the author:

- an approval once the points are fixed;
- a new blocking finding.

Don't post "still open" nudges, don't repeat points already in open threads, and don't add optional nits in a later round. A PR with no code change since the last review gets nothing. When presenting a round to the user, list what you are deliberately not posting.
