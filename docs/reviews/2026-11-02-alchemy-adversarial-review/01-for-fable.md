# FOR: move Kaipu's Cloudflare deploys to Alchemy (advocate brief)

Author: Fable (design sub-agent, assigned FOR). Date 2026-11-02. Sources: `00-context.md` (F#/U#),
plus 8 opened items: hub `monorepos/wrangler-env-config.md`, `effect/why-effect.md`,
`effect/adoption-strategy.md`, `infra/deploy-environments.md`; Kaipu `backlog/deploy-environments.md`,
ADR 0007; https://alchemy.run/environments/ci; https://alchemy.run/state-store.

## Position

I argue reading **(a) full IaC as the end state, reached by a staged path whose first step is the
dev environment Kaipu has already promised itself** (ADR 0007:41-44; F8). End state: one
`infra/alchemy.run.ts` stack declares the three Workers, their assets, custom domains, service
bindings, Access on preview URLs, the two R2 buckets and the `updates.kaipu.app` domain; stages
`prod` and `dev` resolve physical names and bindings per stage; CI runs `alchemy deploy --stage prod`
from the same tag-gated release jobs (F6), with secrets still shipped inside the deploy. `wrangler`
stays for local `wrangler dev` only if U4 resolves well; otherwise it is gone. I do **not** argue
reading (b): replacing `wrangler deploy` while leaving every other resource by hand buys nothing,
and the context package is right that D1/KV/DO/Queue management is irrelevant to Kaipu (F2).

## Core arguments

1. **The dev environment Kaipu already committed to is the exact shape wrangler handles worst and
   Alchemy handles natively.** ADR 0007 states a future dev environment "is a separate Worker (own
   name, own bindings…)" and that "the service binding must resolve per environment, or a dev front
   end writes through the production API" (ADR 0007:41-44). The owner wants that tier long term,
   "as a need for new products, not only this one" (`backlog/deploy-environments.md:67`; F8). The
   hub calls the binding-by-name split "the single most expensive thing to get wrong here" and
   notes wrangler environments do not inherit bindings (`infra/deploy-environments.md:84-93`; F22).
   Alchemy's stage model gives each stage its own physical names and state, with the documented
   `pr-<n>` / `prod` pattern driven by one `--stage` flag (F19; alchemy.run/environments/ci).
   *What changes:* one stack file declares `kaipu-api` and `kaipu-web` once; `--stage dev` produces
   `kaipu-api-dev` and `kaipu-web-dev` with the binding pointing at the dev API without a second
   `wrangler.jsonc` block to keep in sync by hand.

2. **Resources that exist only in the dashboard become code.** Two R2 buckets, the
   `updates.kaipu.app` public domain set under "R2 → Settings → Custom Domains", Cloudflare Access on
   preview URLs, and the `kaipu.app` zone have no declaration anywhere (F7). Alchemy covers every
   one of these types (F15). Kaipu has already paid for undeclared infrastructure once: a design
   session "planned around a migration that did not need to happen" because the repo misrepresented
   how it deployed (F10). *What changes:* `deployment/ci-cd-pipelines.md:116,141` stops being the
   only record of how `updates.kaipu.app` is wired; `alchemy plan` shows drift against the account.

3. **Wrangler version drift is a real, recorded failure in this repo; a single lockfile-pinned
   dependency removes the class.** Three wrangler versions are in play (workspace `4.63.0`, CI
   `4.148.0`, latest `4.156.1`), the pin is not in the Bun catalog contrary to the hub rule, and
   `api-v0.5.0` failed in production with "Unknown arguments: secrets-file" because the action
   resolved a different wrangler than the script expected (F3, F13). The hub's own rule is "pin
   once … and mirror that version in every CI step that installs it globally"
   (`wrangler-env-config.md:31-37`), which is a manual two-place invariant. *What changes:*
   `deploy-worker.sh:21`'s hard-coded `wrangler@4.148.0` and the `bun add -g` mirror go away; the
   deploy tool is a normal workspace dependency resolved from `bun.lock` like every other.

4. **The release gate gets a real plan instead of a dry run.** Release-candidate PRs dry-run the
   deploy script today (F6). Alchemy's plan step diffs declared config against stored state and
   prints create/update/delete per resource (F16: logical-ID changes "plan create-new-then-delete-old").
   *What changes:* `pr-checks.yml:304-312` runs `alchemy plan --stage prod` and the reviewer sees
   "0 to destroy" before approving a tag. This is also the mitigation for argument 2's drift.

5. **Secrets become typed, declared configuration with a direct Infisical path.** Production secrets
   are GitHub Secrets while Infisical holds only `dev` (F5); the `jq` → `worker-secrets.json`
   bridge is eleven keys of shell (F4). Alchemy reads secrets via `Config.Redacted` and documents an
   Infisical provider (package §4 Secrets). *What changes:* the empty-value-drop rule becomes a
   filter in TypeScript next to the declaration, not a `jq` idiom. Caveat: the atomic "secrets ship
   in the deploy" invariant is U3 and is the pilot's primary test (see Risks).

6. **The console Worker gets a deploy path by existing in the stack.** `kaipu-console` has a
   `wrangler.jsonc` with a custom domain and a binding but no workflow at all (F1, U12). In a stack
   file it is three lines beside the other two. Minor, and conditional on U12.

Benefits I do not claim: D1/KV/DO/Queue provisioning (F2: none exist); "wrangler-free dev" (Kaipu's
local loop is `wrangler dev` and nothing is wrong with it); the TanStack Start integration (U6 is a
risk, not a feature).

## Why the prior decision does not bind

The hub rule forbids "a second, redundant deploy path (e.g. an `alchemy.run.ts` orchestrator)
alongside" wrangler, because "two paths that both deploy the Worker drift and one silently wins"
(`wrangler-env-config.md:25-29`). I agree with that rule and propose a **single** path. The rule's
premise is also factually stale for Kaipu: it says CI uses `cloudflare/wrangler-action` and wrangler
is pinned in the catalog; neither is true since `8f14a05c` (F3, F13). A rule whose stated
mechanism no longer exists should be reread, not obeyed by letter.

Commit `52b45368` deleted a v1 (`0.81.4`) scaffold file that "no workflow called" and that misled
readers (F10). That is the removal of dead code, and it was correct. It records no evaluation of
Alchemy as a deploy tool, and the package confirms "it was never evaluated" (§3). Its one
substantive concern, that a laptop `deploy` script "would route around" the verify gate, is
preserved here: Alchemy runs only from the tag-gated CI jobs, there is no local `deploy` script, and
the dev stage is reached by a `workflow_dispatch` job, not from a laptop (F6; ADR 0004 → 0027).

The honest residue: during the pilot, `scripts/deploy-worker.sh` stays as the production path while
Alchemy owns only stage `dev`. That is two tools but not two paths to the same Worker, which is what
the hub rule actually prohibits. It must have a sunset date (see Pilot).

## The Effect angle

What Kaipu would write is a resource declaration file plus `Config.Redacted` reads. It would not
write `Effect<A, E, R>` programs, own a runtime, or export anything with an `R` channel. In the
hub's terms this is below level 1: the runtime owner is the Alchemy CLI, and the boundary never
touches Kaipu code (`adoption-strategy.md:13-17, 23-33`). The hub's warning that popularity-driven
adoption yields the half-adopted trap (`why-effect.md:91-98, 152-155`; F12) targets product code
that imports Effect as a Promise wrapper. Consuming a library whose internals are Effect is not that.

What Effect buys Kaipu: nothing from the hub's list of six payoffs (`why-effect.md:36-61`) accrues
to Kaipu's code. What it buys the owner indirectly: the tool's own retries, interruption and typed
failures, and the state-store backends being swappable `Layer`s (alchemy.run/state-store), which is
how the Postgres option below exists. I will not dress that up as adoption value.

What it costs: when a deploy fails, the stack trace runs "through the fiber runtime" and `Cause` is
"worse than a stack until" you can read it (`why-effect.md:87-89`). The owner will read one. Also,
`effect` is a peer dependency whose "unstable APIs can break in minor releases" (#2764, F14 context),
so `effect` must be pinned exactly alongside `alchemy`. The one place the hub's warning bites for
real: if a resource Kaipu needs is missing or wrong (say the R2 custom-domain setting, U2), writing a
custom resource means writing Effect in a tooling file. That is level 1 in a one-off, which
`why-effect.md:141-143` says is not worth it. If the pilot hits that, it is a concession trigger.

U8 (Effect in the runtime bundle) is presumably no, since the stack file is deploy-time only, but
it is unverified and the pilot must check the built Worker's bundle.

## Risks I accept and how I would bound them

- **State store (F17, U5).** I would not use `Cloudflare.state()` in the pilot. The state-store page
  documents `postgresState()`, "a Postgres database you already run", with a per-`(stack, stage)`
  session advisory lock so two deploys of one stage cannot interleave; it is the only backend that
  documents locking. Kaipu already runs Neon Postgres (F7). A separate database on the same Neon
  project (assumption: Neon allows it; not verified in the package) sidesteps the account-wide
  singleton (#1194), the beta.83 overwrite bug (#2604) and the CI bootstrap refusal. Trade accepted:
  deploys now depend on Neon being up, which production already does. Concurrency is additionally
  bounded by the existing `cancel-in-progress` and serialized preview settings (§2).
- **Destroy by default (F16, U2).** Phase 1 declares only Workers, assets, domains and bindings;
  R2 buckets, Access and DNS stay by hand until phase 4, and when adopted they get an explicit
  `retain` policy. The CI API token for the pilot is scoped to Workers (no R2 or zone delete), so a
  wrong plan cannot delete data even if executed. The destroy job carries the documented
  "fail if STAGE is prod" guard (alchemy.run/environments/ci). `alchemy plan` on RC PRs is the
  human check.
- **Beta cadence (F14, U7).** Pin `alchemy` and `effect` to exact versions in the root Bun
  catalog, which is the rule Kaipu already violates for wrangler (F13). Upgrades go through an RC
  PR whose plan must show zero unintended changes. I accept that a breaking beta can land between a
  pin bump and a hotfix; the bound is the retained `deploy-worker.sh` as break-glass until phase 2
  completes, after which the bound is "do not bump during a release week".
- **Secrets (F18, U3).** This is the risk I cannot bound by design, only by test. The pilot must
  run the exact sequence that produced the `web-v0.9.0` incident (F4): deploy with secrets, upload
  a version without deploying, deploy again, and confirm all eleven keys survive and empty values
  are dropped. #2757 (secrets written outside Alchemy lost across an upload) is the precise hazard;
  the mitigation is that every secret goes through the declaration, none through the dashboard.

## Steelman of the opposition

1. **"YAGNI. Kaipu is three Workers, three domains and two bindings (F2). Wrangler does that in
   twelve-line files. You are importing a state store, a beta dependency and a destroy footgun to
   solve a dev environment that ADR 0007 explicitly deferred until a writer appears."**
   Rebuttal: the dev tier is not speculative; the owner said yes, long term, and named the reopen
   triggers (F8; `backlog/deploy-environments.md:119-127`). Building it later with wrangler means
   hand-duplicated bindings and a by-name service binding that the hub itself flags as the costliest
   mistake (F22). Where this rebuttal is weak: if the trigger fires in 2028, the right time to adopt
   is 2028, and my proposal becomes conditional on timing. I accept that.
2. **"A beta that ships every four days, says 'expect breaking changes', and had a state-corrupting
   bug six days before this review (F14, F17) is not what you put under a product with real
   users."** Rebuttal: pinned exact versions, Postgres state instead of the affected store, the RC
   plan gate, and fewer than ten users (F8) bound the blast radius; the dev-stage pilot touches no
   production object at all. Weak part: a hotfix during a broken beta needs the retained wrangler
   script, which is the transitional two-tool state the hub rule dislikes.
3. **"The secrets invariant was earned by a production incident (F4). Alchemy has four open
   secret-lifecycle bugs (F18) and U3 is unverified. You are proposing to replace the one part of the
   pipeline the repo learned the hard way."** Rebuttal: the pilot's pass/fail criterion is exactly
   this sequence, run on a stage with no production exposure. That is a plan to find out, not an
   answer. This is the strongest opposing argument and my rebuttal is weak by construction: if the
   test fails, FOR is wrong.
4. **"Cloudflare's first-party tools already give per-branch previews and auto-provisioning
   (F20)."** Rebuttal: Workers Builds is git-connected and bypasses the release-please tag gate
   (F6); wrangler environments do not solve the binding-by-name split (F22). Fair rebuttal, but it
   only matters once the dev tier exists, which loops back to point 1.

## What would make me concede

- U3 resolves badly: the deploy → version upload → deploy sequence loses or blanks any secret, or
  secrets cannot ship in the same operation as code. Immediate concession.
- U4 resolves badly: v2 cannot emit or consume a `wrangler.jsonc`, so `wrangler dev`, `wrangler
  types` and `versions upload` require a hand-maintained parallel config. Two sources of truth is
  the hub rule's actual concern, and reading (b) is not worth adopting.
- U6 resolves badly: the web build under Alchemy's Vite integration differs from `bun run build`
  + wrangler bundle. Concede for `kaipu-web`; `kaipu-api` alone is not enough to justify the move.
- U1 resolves badly: adopting `kaipu-web` requires DNS or certificate re-issuance on `kaipu.app`.
- Timing: no reopen trigger from `backlog/deploy-environments.md:119-127` fires by 2027-11-01.
- Stability: v2 is still beta on 2027-04-13 (two years after beta.1), or two pinned-version bumps
  in a row require stack-file changes.
- Backing (U11): no release on `alchemy-run/alchemy` for 90 consecutive days.
- Cost: the pilot below exceeds three working days, or the pilot needs a custom Effect resource.

## Proposed pilot

**Smallest reversible experiment: stage `dev` of a new stack, touching no production object.**

- *Where:* `infra/alchemy.run.ts` at the repo root declaring `kaipu-api` and `kaipu-web` (console
  excluded until U12 is answered), with physical names `kaipu-api-dev` / `kaipu-web-dev`,
  `workers_dev` on, no custom domains, service binding resolved per stage. State in
  `postgresState()` on a dedicated database. `alchemy` and `effect` pinned exactly in the catalog.
- *Trigger:* a `workflow_dispatch` job `deploy-dev.yml` with a `ref` input, mirroring
  `preview-web.yml`'s build steps (F3 table) and a GitHub Environment `dev` holding non-production
  secrets. Nothing on tags changes; `release-*.yml` are untouched.
- *Must prove, in order:* (1) U3 secret sequence passes with all eleven API keys, empty values
  dropped; (2) U6: `dist/client` and Worker bundle byte-comparable to the wrangler build, and no
  `effect` in the runtime bundle (U8); (3) the dev web Worker calls the dev API, never `kaipu-api`
  (F22); (4) `alchemy plan` on a second run reports no changes; (5) `alchemy destroy --stage dev`
  removes both Workers and nothing else.
- *Rollback:* `alchemy destroy --stage dev`, drop the database, delete the file and workflow. No
  production resource was referenced, so rollback is a revert.
- *Sunset:* if (1)-(5) pass, phase 2 adopts `kaipu-api` prod with `--adopt` only after a plan shows
  zero diff, and `deploy-worker.sh` is deleted within one release cycle after `kaipu-web` moves.
  If any step fails, the conceding condition above applies and the pilot is reverted, with the
  result recorded in `backlog/deploy-environments.md` per the hub's "record even when the decision
  is not yet" rule (`infra/deploy-environments.md:110-117`).
