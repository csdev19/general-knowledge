# Where the CI runner comes from

_[ci-per-project-pipelines.md](./ci-per-project-pipelines.md) decides **how often** a suite runs.
[ci-runner-cost.md](./ci-runner-cost.md) decides **what each minute costs**. This page decides
**who owns the machine** — GitHub, a managed third party, or you. It is the axis people reach for
first and should reach for last._

## Three provenances

| Provenance             | You pay for                        | You operate            | Typical reason to pick it              |
| ---------------------- | ---------------------------------- | ---------------------- | -------------------------------------- |
| GitHub-hosted          | minutes, at GitHub's multipliers   | nothing                | default; clean environment per job     |
| Third-party managed    | minutes, at the vendor's rates     | nothing                | faster hardware without owning a fleet |
| Self-hosted            | electricity and your own attention | the machine, forever   | an always-on machine you already have  |

Switching provenance is a one-line change in `runs-on:`. Everything expensive about the decision
is downstream of that line.

## What third-party managed runners actually sell

Not a lower price per minute. Rates as published in September 2026, against GitHub's
$0.006/min Linux and $0.062/min macOS (GitHub cut hosted-runner prices by up to 39% on
2026-01-01, so any comparison written before that is stale):

| Provider   | Linux                            | macOS                          | Fixed fee     | Billing granularity      |
| ---------- | -------------------------------- | ------------------------------ | ------------- | ------------------------ |
| Depot      | $0.006/min (2 vCPU)              | $0.08/min (8 vCPU, 24 GB)      | $20/mo        | per second, no minimum   |
| Blacksmith | $0.004/min x64, $0.0025/min ARM  | $0.08/min (M4)                 | none          | per minute, rounded **up** |
| Namespace  | $0.004/min prepaid, $0.006 overage | $0.06 prepaid, $0.09 overage | none, or $100/mo for prepaid rates | per second |

Three things that are not on the pricing pages:

**1. On macOS, every one of them is at or above GitHub.** The saving they advertise there comes
from finishing sooner on better hardware (6–8 cores against GitHub's 3), not from the rate. That
only pays if the macOS job is actually CPU-bound — see the next section.

**2. Billing granularity can matter more than the rate.** A repository whose PR gate is a
three-second reporting job pays a **full minute per job** on GitHub and on Blacksmith, and three
seconds on Depot or Namespace. When a workflow fires many tiny jobs, this term dominates the
rate entirely.

**3. All of them require a GitHub organization.** This is the constraint that kills the option
outright for a personal repository, and none of them lead with it:

- Blacksmith: _"limited to GitHub organizations and not available for personal repositories."_
- Depot: _"you must have the organization owner role."_
- Namespace: _"you need to connect Namespace with your GitHub organization."_

Moving a personal repository into an organization to satisfy this is not free either: a Free
organization includes **2,000** minutes a month where a Pro personal account includes **3,000**.
If the reason to move was cost, price the quota loss into the comparison.

Two more facts worth carrying:

- **Vendor minutes do not consume GitHub's included quota** — they are billed by the vendor from
  minute zero, while the included allowance sits unused. A repository comfortably inside its
  included quota has nothing to save by leaving it.
- **This market consolidates.** BuildJet and Cirrus CI both shut down in 2026. A `runs-on:` label
  is cheap to change back, which is the mitigation; a workflow built around one vendor's
  proprietary cache or disk feature is not.

One pricing scare to check rather than remember: GitHub announced a **$0.002/min "Actions cloud
platform charge" on self-hosted runners in private repositories** in December 2025, and postponed
it a day later after pushback. It has never applied to third-party runners. Re-read the current
billing docs before deciding, because if it ever lands it changes the self-hosted case, not the
managed one.

## When a faster runner is the wrong purchase

A faster runner only shortens a job that is **bound by the runner's own CPU or disk**. Before
paying for one, check whether the time is going somewhere else. Four bottlenecks that look like
slow CI and are immune to faster hardware:

1. **A wait on an external service.** Code signing and notarization, a deploy that polls for
   readiness, a queue. Apple's notarization is minutes of network wait; a twelve-core Mac waits
   exactly as long as a three-core one. The lever here is **fewer round-trips or off the critical
   path**, not more cores. A build that notarizes two architectures sequentially pays twice.
2. **A cache that never hits.** Vendors advertise "4× faster cache downloads". A cache that misses
   is not slow, it is absent, and 4× nothing is nothing. Diagnose the hit rate first — the
   read-side ref scoping in [actions-cache-lifecycle.md](./actions-cache-lifecycle.md) makes
   release-only pipelines miss by construction.
3. **Per-job minimum billing.** Jobs of a few seconds cost a whole minute each on GitHub. Faster
   hardware cannot go below the minimum; deleting the job or merging it into another can.
4. **Work that should not run at all.** A transitional alias job, a bot on every docs merge, a
   suite that path filters should have excluded.

The honest test: take the slowest job, list its steps with their durations, and mark each one
`cpu`, `io`, `wait` or `overhead`. If `cpu` is not the largest column, a faster runner is not the
purchase.

## Self-hosted: installation is the easy part

Registering a runner takes about half an hour per machine: download `actions/runner`, run
`./config.sh --url <repo-or-org> --token <token>`, then `./svc.sh install && ./svc.sh start`. The
runner **polls GitHub outbound over HTTPS** — no inbound ports, no public IP, no tunnel, no
reverse proxy. Most of what people expect to be hard about this does not exist.

What you take on instead:

1. **The machine image, forever.** GitHub's hosted images ship hundreds of preinstalled tools at a
   documented version. Yours ships what you installed. Every implicit assumption in a workflow —
   a codec binary, an Xcode version, a system library, a language runtime — becomes a thing you
   maintain, and the drift is **silent until a release fails**. Write the image down as code
   (a script, a Brewfile, a Dockerfile) on day one, or you will be reconstructing it from memory
   during an incident.
2. **Liveness.** A hosted runner is always there. A machine that is asleep at tag time gives you a
   job queued forever, which reads as a hung release rather than a failed one. Decide the fallback
   explicitly: a second label, a manual dispatch onto hosted runners, or an accepted delay.
3. **State.** See the next section; this is the real trade.
4. **Secrets on hardware you own.** Signing certificates and API keys land in the runner's
   workspace environment. On a private single-owner repository that is an acceptable risk written
   down; it is still a risk that hosted runners did not have.
5. **Never a persistent self-hosted runner on a public repository.** A fork's pull request can run
   arbitrary code on it. This is GitHub's own warning and it is not negotiable.

### Ephemeral or persistent — decide per job, not per fleet

| Mode                                 | You get                                                         | You lose                                                  |
| ------------------------------------ | --------------------------------------------------------------- | --------------------------------------------------------- |
| **Persistent** (one long-lived runner) | Warm `node_modules`, warm package store, warm build cache — the single biggest speedup available | The clean-environment guarantee; one job can poison the next |
| **Ephemeral** (`--ephemeral`, fresh VM or container per job) | Isolation close to a hosted runner                              | Cold every time, plus VM boot; you are paying for hardware to redo work |

These two are in direct conflict, and the conflict maps onto the trigger ladder. An iterative
test job wants persistence — that warm state is exactly the win. A **release gate exists to be
clean**: its whole purpose is to catch a stale lockfile or a dependency that only fails on a fresh
install, and a persistent runner is the one environment guaranteed not to catch it. GitHub has
recommended `--ephemeral` as the default since March 2026 for this reason.

The defensible split for a small setup: **iterative and test jobs on a persistent self-hosted
runner, release and publish jobs ephemeral or left on hosted runners.** If a release gate moves to
a persistent machine, say plainly in the ADR that the clean-environment guarantee was traded away,
and what now provides it.

### Tooling, by what you are trying to run

| Need                                            | Project                                                                 |
| ----------------------------------------------- | ----------------------------------------------------------------------- |
| One Linux machine, simplest thing that works    | `actions/runner` directly, with `--ephemeral`                           |
| Linux jobs isolated per run on one machine      | a container runner such as `myoung34/docker-github-actions-runner`      |
| A Linux fleet that autoscales                   | Actions Runner Controller (ARC) — GitHub's Kubernetes operator; worth it past roughly three or four runners, overkill below that |
| Ephemeral macOS on Apple Silicon                | Tart (Apple Virtualization framework), with Tartelet or Cilicon to wire VMs to the Actions queue |
| A managed macOS fleet                           | Anka + Anklet (Veertu), on your Macs or cloud Macs                      |
| Your own cloud account, someone else's control plane | RunsOn, or WarpBuild's self-hosted mode — you keep the bill and the data, they keep the orchestration |
| Debugging a workflow without pushing            | `nektos/act` — a local approximation, **not** a runner; it does not reproduce a hosted image |

macOS has a licensing constraint the others do not: macOS VMs may only run on Apple hardware, and
Apple's licence permits two VMs per host. A Mac is not optional and a Mac is not elastic.

### Which machine to volunteer

- **Always-on beats fast.** A desktop or a mini that never sleeps is a better runner than a
  faster laptop that closes, moves and suspends. Laptops make poor runners for liveness reasons,
  not performance ones.
- **A desktop CPU against a 4-vCPU hosted runner is a large, free gap.** This is where a
  self-hosted Linux runner wins without any cleverness.
- **A GPU changes coverage, not cost.** It matters only if the suite needs hardware
  encode/decode or CUDA. When it does, the win is not speed: it is that tests which **self-skip**
  on a hosted runner — a codec-dependent export, a hardware-accelerated path — finally run. Read
  that together with the platform-gate trap in [ci-runner-cost.md](./ci-runner-cost.md): a suite
  that skips silently reports green while asserting nothing, and giving it real hardware is one of
  the few ways to fix that rather than hide it.

## Decision order

Do these in order. Each one is cheaper than the one after it, and each can make the next
unnecessary.

1. **Delete work that should not run** — transitional jobs, bots on irrelevant paths, suites the
   path filters should exclude.
2. **Make the cache hit**, and verify it by reading restore times, not by trusting the config.
3. **Remove per-job rounding waste** — merge or drop jobs that cost a minute to take three
   seconds.
4. **Attack the external waits** — fewer signing round-trips, deploys off the critical path.
5. **Only then buy hardware.** Self-hosted if you already own an always-on machine and the
   repository is private; third-party managed if you are an organization and want to own no
   machine. If steps 1–4 brought usage back inside the included quota, stop: there is no bill
   left to optimize.

## Measured example

A private, single-owner Turborepo monorepo (web + API on Workers, an Electron desktop app),
one week, 243 workflow runs, 284 billed jobs:

| Bucket                               | Jobs | Real seconds | Billed minutes | Linux-equivalent |
| ------------------------------------ | ---- | ------------ | -------------- | ---------------- |
| Desktop release (macOS)              | 3    | 1,225        | 21             | 210              |
| Release-candidate E2E (macOS)        | 7    | 780          | 16             | 160              |
| PR gate + transitional alias (Linux) | 81   | 1,912        | **106**        | 106              |
| Release bot (Linux)                  | 44   | 2,080        | 59             | 59               |
| `verify` on candidate and release    | 21   | 1,880        | 49             | 49               |

Total 450 Linux + 37 macOS billed minutes ≈ 820 Linux-equivalent, about $5 at list price.
Reading it against the four bottlenecks above:

- **The 81-job row is 32 real minutes billed as 106.** Roughly 70% of it is the per-job minimum,
  paid by a required-check job and a transitional alias that both take three seconds. No runner
  is fast enough to fix that; deleting the alias halves it.
- **The most expensive single step was 387 seconds of "sign, notarize & package"**, running
  `electron-builder --mac --arm64 --x64` — two sequential notarization round-trips to Apple. Pure
  external wait. All three managed vendors charge **more** per macOS minute than GitHub, and none
  of them would have shortened it.
- **`verify` took 361 s in the cloud against 27 s cold on the developer's machine.** Not a CPU
  gap: the Turbo cache entry existed, was 655 KB, and was never restored, because every entry was
  scoped to a release tag's ref and nothing ever wrote one on the default branch.

The provenance change that looked like the answer addressed none of the three. The order in
"Decision order" is the order those findings imply.

## Related

- [ci-runner-cost.md](./ci-runner-cost.md) — the multiplier and trigger-ladder half of the bill,
  and the platform-gate trap that makes a cheaper runner look green.
- [actions-cache-lifecycle.md](./actions-cache-lifecycle.md) — why the cache misses, on both the
  write and the read side.
- [release-gated-verification.md](./release-gated-verification.md) — what a release gate is for,
  which is what a persistent self-hosted runner trades away.
- [pr-checks.md](./pr-checks.md) — required checks, and why a gate job must report exactly once.
- [../conventions/ci-cd-pipeline-strategy.md](../conventions/ci-cd-pipeline-strategy.md) — the
  cost levers, including the self-hosted runner as the last one.
