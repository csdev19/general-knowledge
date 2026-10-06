# Deploy environments: what a tier costs, and where it goes

Adding a staging environment feels free at design time and is not. It is a cost paid on every
deploy, for as long as the product lives. This page is about deciding whether to pay it, where
the tier goes in the branch flow, and the Cloudflare Workers mechanics that make the choice a
real decision rather than a configuration detail.

## Name what it protects before you build it

A pre-production environment protects users of production from a bad release. So the first
question is not "staging or previews" — it is **how much is a bad release costing today**.

With no users, or a handful who know the product is early, the tier has nothing to protect and
you are buying:

- a second set of secrets, which is usually a second environment in the secret manager with
  every folder duplicated;
- a second copy of every binding, route and variable;
- a step people skip exactly when it matters, because the pressure that makes you skip it is the
  same pressure that produces bad releases.

That is not an argument against ever having one. It is an argument for writing down **the
condition that turns it from overhead into value**, and building it when the condition is met
rather than when it first occurs to you. Good triggers: a second person testing releases, a
measurement series that needs a fixed address over months, a user count where one bad hour costs
more than the tier does.

## The branch releases are cut from must stay releasable

The common first idea is "let `main` be the broken one, and cut releases from somewhere else" —
or its twin, "`main` is dev, we can break it freely".

It does not survive contact with tag-based releases. If tags are cut from `main` and `main` is
where anything may be broken, then a tag is no longer a trustworthy cut: every release becomes a
bet on what landed since the last one, and the thing the tier was supposed to buy — confidence
at release time — is exactly what it destroyed.

So the breakable environment goes somewhere that is **not** the branch releases are cut from.
Two shapes do that, and they compose:

1. **Pre-merge.** `branch → ephemeral environment → main → tag → prod`. One environment per
   branch or PR, torn down with it. `main` stays always-releasable and tags keep their meaning.
   Nothing persists between deploys, so nothing drifts.
2. **Post-merge.** `branch → main → staging → tag → prod`. Every merge auto-deploys to a
   persistent staging; the tag promotes what was already seen there. This is what you want when
   the value is seeing *accumulated* changes together, or when something needs a fixed address.

Shape 1 answers "is this change good". Shape 2 answers "is this release good". They are
different questions, and a product usually needs the first long before the second.

## Cloudflare Workers: three mechanisms, not one

Cloudflare's own guidance moved; designing from older material picks the wrong one.

| Mechanism | Persistence | What it is for |
| --- | --- | --- |
| **Previews** (`wrangler preview`) | ephemeral, per branch | Cloudflare's stated recommended way to test changes before production. Own vars, secrets and bindings; custom domains supported. This is shape 1. |
| **Wrangler environments** (`deploy --env`) | persistent, deploys a `name-env` Worker | when an environment needs persistent Workers with different settings, routes or domains. This is shape 2. |
| **Version URLs** | ephemeral, per version | inspecting one uploaded version before promoting it. The docs say explicitly **not** to use these for branch or PR testing. An upload leaves the Worker's latest version undeployed, and from then on any separate secrets edit is refused — the release has to ship its secrets inside the deploy ([worker-secrets-with-deploy](../monorepos/worker-secrets-with-deploy.md)). |

Two mechanics worth knowing before choosing:

**Bindings and vars are not inherited between Wrangler environments.** Each environment
redeclares them. The failure this produces is the nastiest kind: a binding added to production
and forgotten in staging makes staging break in a way that does not resemble production, so the
environment actively misleads you at the moment you trusted it.

**Service Bindings bind to a Worker by name.** A front end bound to `my-api` keeps talking to
`my-api` no matter which environment the front end is in. Unless the binding target is resolved
per environment, a staging front end talks to the production API — and writes to its database.
Any environment design that splits a bound pair has to answer this explicitly; it is the single
most expensive thing to get wrong here.

Sources: [Compare workflows](https://developers.cloudflare.com/workers/previews/compare-workflows/) ·
[Introducing Worker Previews](https://blog.cloudflare.com/worker-previews/) ·
[Workers best practices](https://developers.cloudflare.com/workers/best-practices/workers-best-practices/)

## Splitting one half of a bound pair is a decision, not a shortcut

"We only need the front end in staging, the API is fine in production" is a reasonable thing to
want and a dangerous thing to implement silently. It means a pre-production front end issuing
real writes against real data.

It is defensible when the thing being tested genuinely does not write — a marketing page, a
render-only measurement — and indefensible as a general setting. Write down which it is. If the
answer is "only for now", the binding must still be resolved per environment, so that the day
the test does write, it writes somewhere harmless.

## Record the decision even when the decision is "not yet"

A design conversation that ends in "we decided to wait" produces the same research as one that
ends in a build, and loses it just as fast. Capture it: what the repository already does, what
the state of the art does, the options with their counterarguments, the answers given, and the
condition that reopens it. A later session then writes the spec from the record instead of
re-running the investigation — and the reopen condition is what stops the question from being
reargued from zero every few months.
