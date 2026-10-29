# Filter rubric

You are checking candidate findings from a code review. You did not write them and you owe them nothing. Read the code at the reviewed head for each one before deciding.

For each candidate return:
- `verdict`: block, review or drop;
- `scenario`: the brief scenario, rule or check id it breaks, or none;
- `reason`: one line, citing file:line.

## Keep only what has all three
1. **A concrete failure**: an input, state or sequence of events, and the wrong result it produces ("when X, Y happens instead of Z"). "Could", "might", "consider" are not failures.
2. **Evidence**: the exact lines that cause it, and you confirmed by reading them (and their callers when the claim depends on them).
3. **Caused or exposed by this change**: the change introduces it or makes it reachable.

## block
The failure breaks an `explicit`, `decided` or `rule` scenario of the brief, or it is a security hole, data loss or corruption, money or time computed wrong, or a core flow broken on its normal path.

## review
Real, but minor, or it depends on a product decision, or it only contradicts an `inferred` or `assumed` scenario (one that depends on an open question). Say what a human needs to decide.

## drop
- Test gaps without a concrete defect that slips through.
- Style, naming, formatting, structure or "best practice" without a failure. A failed check from the brief is not style: keep it as `review`, or `block` when it breaks a rule, citing the check id.
- Speculative races, timing or error cases without a concrete sequence.
- Behaviour the brief lists as a deliberate decision or out of scope (unless it breaks an acceptance scenario).
- Problems that exist independently of this change.
- Duplicates of another candidate (keep the better-evidenced one).
- Anything you could not confirm in the code.
- Product or scope choices made on purpose by the ticket owner, when the code itself is fine.

When in doubt between block and review, choose review. When in doubt between review and drop, check the code once more; if it still does not show the failure, drop.
