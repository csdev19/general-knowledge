# Test audits

**A test earns its place by catching a credible failure that nothing else
catches.** Agents write many tests, and most restate the code they were
written after: they always pass, break on every refactor, and catch nothing.
An audit applies the bar in both directions: delete what cannot fail, and add
what is missing for the failures the owner actually fears.

The executable version of this convention is the
[`cs-audit-tests` skill](https://github.com/niway-dev/skills/tree/main/skills/cs-audit-tests). This page is the reasoning; the
skill is the procedure. The junk patterns and retention bar descend from
OpenClaw's `test-audit` skill; the gap hunt is our addition.

## Two families of junk

**Tautological** tests cannot fail: the assertion is true by construction.
Assertion-free probes, self-comparisons, constants restated, source greps,
mocks that implement the behaviour being asserted, expected values produced by
the helper under test.

**Change-detector** tests fail only when the code changes, not when it is
wrong. A unit test that mirrors the function body line by line is one; so is a
regression test for a fix an existing behaviour test already covers.

Both are cheap to write and expensive to keep, because an agent burns the
next session repairing them instead of the feature.

## The bar cuts both ways

Deleting is only half the audit. Before touching anything, write down the
failures that matter in this project (money, data loss, auth, silent
corruption, a third-party contract) and find each one's owner test. A failure
with no owner is a gap, and a gap is worth more than ten deletions.

Tests written from the code restate it. Tests written from the failure list
catch it. So the failure list comes **before** the test, whether the test is
new or being repaired.

## Rules that carry the value

**Judge by assertions, not by name.** A test named for retiring a window that
asserts the window was *not* cleared is a change-detector with a good name.

**One owner per failure.** The test lives at the boundary that owns the
behaviour, once. A caller earns its own test only for a failure the owner
cannot see.

**A regression test must fail first.** Run it against the unfixed code and
quote the failure. A test that never failed proves the mock.

**A defect's semantics belong to the owner.** Clamp or reject, both pass the
test you would write for the other. Propose, then fix.

**Slow or static is not a deletion reason.** A test that looks like
implementation may still be the cheapest independent guard of a contract;
prove otherwise before removing it.

## Related

- [verifiable-handoffs](./verifiable-handoffs.md): the audit's report
  quotes the check's exit code and separates production from test lines.
- [ci-cd-pipeline-strategy](./ci-cd-pipeline-strategy.md): the `verify`
  gate the retained suite runs under.
