# Repository families: separate repositories, one workspace for agents

> Status: **tested 2026-11-13** on one product split into a public and a private repository
> (example names here are invented: `acme-engine` public, `acme-cloud` private,
> `acme-workspace` the coordinator). The cost of the injected instructions is not measured yet.

A **repository family** is a set of separate repositories that an AI agent treats as one
virtual monorepo. A private **coordinator** repository holds the shared instructions and the setup
script. The members are cloned inside it and ignored by git. This pattern has been shared publicly
by other teams; this page is one concrete, tested version of it.

## The problem

An agent works in one repository and does not know its siblings exist. When a product is split into
a public repository (what users run themselves) and a private one (what the owner hosts), the agent
can write private code in the public repository and push it.

- **A push to a public repository is a publication.** Deleting the branch afterwards does not remove
  the commits: they stay reachable by hash until the host's support purges them.
- **A note in an assistant's memory does not fix it.** It lives on one machine, nobody reviews it, and
  nothing enforces it.
- **A real monorepo is not an option**, because one repository cannot be half public. Submodules were
  not wanted.

## Layout

```text
acme-workspace/          private coordinator repository
  AGENTS.md              shared instructions
  CLAUDE.md              one line: @AGENTS.md
  .gitignore             acme-engine, acme-cloud   (no trailing slash)
  .ignore                !/acme-engine/  !/acme-cloud/
  setup.sh               clones members, sets config, installs hooks
  hooks/                 session hook and pre-push logic, allow list
  acme-engine/           public member, a real clone
  acme-cloud/            private member, a real clone
```

## The pieces and why each exists

1. **`AGENTS.md` in the coordinator.** It holds:
   - the members: directory, repository, visibility, what it holds;
   - the one-sentence rule for what is public;
   - a "where things go" table with concrete pieces;
   - a "before you change X, read sibling Y" table;
   - a "never" list: a public member never names a private one, no `--no-verify`;
   - how to work.

   `CLAUDE.md` is a single line, `@AGENTS.md`, because Claude Code reads `CLAUDE.md` and reads
   `AGENTS.md` only when no `CLAUDE.md` exists.

2. **`.gitignore` and `.ignore`.** `.gitignore` lists the member folders without a trailing slash.
   `.ignore` re-includes each with `!/<member>/`, because ripgrep skips gitignored folders and a
   search from the root would otherwise find nothing. Each member's own `.gitignore` is still
   honoured.

3. **Members are real clones, not symlinks.** Ripgrep does not enter symlinked folders without `-L`,
   and a `.gitignore` entry with a trailing slash does not ignore a symlink.

4. **A local git config key per member.** Run
   `git config --local family.coordinator <coordinator path>` in each member checkout. Local config
   is shared by all worktrees of a checkout, and nothing is committed into the member, so a public
   member never names the coordinator or a private sibling.

5. **A Claude Code session hook.** Register it on `SessionStart` (sources `startup`, `resume`,
   `clear`, `compact`) and on `SubagentStart`. It reads the key from the session's `cwd` and injects
   the coordinator's `AGENTS.md` as `additionalContext`.
   - Keep the injected text under about 8,500 bytes. Claude Code saves `additionalContext` over
     10,000 characters to a file and shows a 2,000-character preview instead
     ([hooks docs](https://code.claude.com/docs/en/hooks)).
   - Read the key with `--local`, so a global value does not make every repository a member.
   - The same hook warns when a checkout of a known family is not linked, or the coordinator is
     missing, and offers the setup commands without running them.

6. **A git `pre-push` hook in each public member.** The logic lives in the coordinator. The hook:
   - reads the allow list from the coordinator's `origin/main`, never its working tree;
   - checks every commit being pushed that the remote does not have;
   - refuses a path whose top-level entry is not in the allow list, or a deploy file name (for
     example `wrangler.toml`, `fly.toml`, `vercel.json`, `netlify.toml`);
   - refuses when the config key is unset or the list is unreadable;
   - refuses a push from a public member to a destination outside the family.

   Replaying two real leaky pushes through it: both refused; an ordinary push passed. **Limits:** it
   is local, so `--no-verify` or editing the script bypasses it, and it does not inspect file
   contents.

7. **`setup.sh`.** It clones missing members inside the coordinator, sets the key in each, and
   installs the pre-push wrapper in public members (it stops if `core.hooksPath` is set or another
   `pre-push` exists). With `--also-dir <parent>` it links checkouts that already exist elsewhere.

## Set up a new family

By hand, in order. The steps can be scripted.

1. Create the private coordinator repository and clone it as `acme-workspace`.
2. Add `AGENTS.md` (members, the public rule, where-things-go, read-the-sibling, never, how to work)
   and `CLAUDE.md` containing only `@AGENTS.md`.
3. Add the member folders to `.gitignore` without a trailing slash, and the `!/<member>/` lines to
   `.ignore`.
4. Clone each member inside the coordinator:
   ```bash
   cd acme-workspace
   git clone <acme-engine url> acme-engine
   git clone <acme-cloud url> acme-cloud
   ```
   If a clone already exists elsewhere, move it in (see the worktree note below) rather than
   symlinking it.
5. In each member, set the key:
   ```bash
   git -C acme-engine config --local family.coordinator "$PWD"
   git -C acme-cloud config --local family.coordinator "$PWD"
   ```
6. Write the allow list of top-level paths for each public member, and the pre-push logic, in the
   coordinator. Commit and push the coordinator.
7. In each public member, install a `pre-push` hook that calls the coordinator's logic. Do not
   overwrite an existing `pre-push` or a `core.hooksPath`.
8. Register the session hook in the Claude Code settings on `SessionStart` and `SubagentStart`.
9. Verify (below).

## Working with a family

- **Day to day,** work in the member's own project and worktrees. The hook gives every session and
  subagent the family rules.
- **Cross-repository questions** are asked from the coordinator root, where search reaches every
  member.
- **Never make a worktree of the coordinator.** It carries none of the members.
- **A change that spans members** is decided once, then built as one issue and one pull request per
  member, cross-linked. Editing two members in one worktree is not solved by this pattern.
- **Moving an existing repository into the coordinator** breaks its worktrees. `git worktree repair`,
  or a symlink left at the old path, fixes them.
- **Verify after setup:** from the root, ask a question that needs two members and check that the
  answer cites files from both.

## Cost

The injected instructions add input tokens to every session and subagent. Measure it before and
after with your usage tool; the figure is "measure it" until a measurement is recorded here.

## Limits

- The pre-push hook is local: `--no-verify` or editing the script bypasses it. A server-side check
  in the public repository would be the enforceable layer; this pattern does not provide one.
- It inspects paths, not file contents.
- Nothing here fixes two members edited in one worktree.

## Related

- [Tool doctor](./tool-doctor-pattern.md) — the same "check and report, offer the commands, do not
  run them" stance the session hook takes.
- [Repo briefings](./repo-briefings.md) — how a single repository explains itself to its readers.
