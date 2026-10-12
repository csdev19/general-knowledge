# Publishing technical writing drafted with AI

**Quote the author verbatim, draft the rest with an AI, and show the reader
which is which.** A post written end to end by a model reads as such within two
paragraphs, and a reader who notices stops trusting the content, however good
it is. Writing everything by hand does not survive a weekly cadence. The split
keeps both: the author owns the claims a reader would want a person behind,
and the model does the expensive, mechanical part.

Skills: `cs-write-blog-post` (procedure) and `cs-audit-blog-post` (review), in
the [skills repo](https://github.com/niway-dev/skills).

## The two kinds of text

| Text | Written by | Contains | Bar |
| --- | --- | --- | --- |
| Author blocks | the author, in a raw input file, then quoted | the hook (what hurt, what changed, why), opinions, "what I'd do differently" | the author's voice; only spelling and grammar may change |
| Drafted body | a model, from the source repository, reviewed by the author | the mechanism, code, numbers, trade-offs | technically correct and clear; density is fine |

The author's raw input lives in a file next to the post that the site never
publishes (`human.md` beside `index.md`, one `## <section>` per block). The
post quotes each section inside a marked block. The marker must render, so
the reader sees it; a comment-only marker defeats the purpose.

A verifiable rule beats a promise: a lint compares every author block with
its source section and fails when the word-level similarity drops below a
threshold (0.85 works: it allows typo and grammar fixes, and it fails a
rewrite). The lint cannot prove who wrote the raw file; that stays the
author's discipline.

## Disclosure

Every post with author blocks carries one line that says what the marker means
and how the rest was produced ("drafted with <model> from the source
repository and reviewed by me"). Platforms such as DEV already require this
kind of disclosure for AI-assisted posts, and the W3C AI Content Disclosure
community group is defining structured authorship levels. A disclosure is
only useful when it is specific: a marked block tells the reader which
sentences the author stands behind personally.

## The interview

The author answers, in the publication's language and without polishing:

1. **hook**: What hurt, concretely? What did you change? Why did you build it
   instead of using what existed?
2. **lesson**: What surprised you, or what would you do differently today?
3. Optional: an opinion the post argues for, or a mistake you made.

Answers go into the raw file as written. Asking the model to "improve" them is
the failure this whole convention prevents.

## Style limits for the drafted text

These are the tells readers recognise. They can be measured, so a script
enforces them, not a reviewer's memory:

| Tell | Limit |
| --- | --- |
| Em dashes | at most one per 200 words |
| Contrast reveal ("X isn't A. It's B.") | none |
| Self-labelling sections ("Honesty section", "the post I wish I'd found") | none |
| Stock phrases ("delve", "let's dive in", "it's worth noting", "game-changer") | none |
| Bold spans | at most one per section (warning) |
| Placeholders (`TODO`, `TBD`) | none in a published post |

Beyond the measurable list, a reviewer checks for things a script misses:
every claim is backed by a commit, file or measurement; each section makes
one point; numbers carry their unit and source.

## Topics and cadence

Material comes from work that already happened: merged pull requests, ADRs,
measurements. A weekly harvest of the author's merged PRs across repositories
proposes topics. A topic that needs more than one post becomes a series, cut
where a reader could stop and still have something usable.

## Measured example: cs19.dev, 2026-10-03

Before this convention, the lint on three published posts drafted end to end
with a model found 11 errors: no author block in any of them, em dashes at 2–4
times the limit (20 in 1,187 words in one post), two contrast reveals and two
self-labelling phrases. After rewriting one post to the model, only the two
author blocks waiting for the author's input remained.

## Related

- [adr-pattern.md](./adr-pattern.md): the decision to adopt this in a project is an ADR.
- [verifiable-handoffs.md](./verifiable-handoffs.md): the same principle of checks over claims.
- [../product/product-marketing-playbook.md](../product/product-marketing-playbook.md): claims and proof for product pages.
