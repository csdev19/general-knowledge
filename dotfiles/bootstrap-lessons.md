# Dotfiles Bootstrap: What Breaks the First Time You Run It On Real Hardware

_A single `install.sh` that clones a dotfiles repo and reproduces a whole
workstation reads as a solved problem — until it runs on an actual second
machine. Every failure below was found by running the real script on real
hardware, not by reviewing the code._

---

## The core problem: bulk package managers are not bootstrap-shaped

`brew bundle install` (and the equivalent for `apt`, `winget`, etc.) treats a
whole package list as one atomic operation. That is a poor fit for a bootstrap
script, for two independent reasons:

1. **One bad entry kills observability into the rest.** A single cask needing
   an interactive `sudo` password (no TTY available), an app the App Store
   isn't signed into yet (`mas`), or a formula from an untrusted third-party
   tap, makes the whole command return non-zero — you cannot tell from the
   exit code which of the other 30 entries actually succeeded.
2. **It is not resumable.** Re-running the same bulk command after fixing one
   blocker re-attempts (or re-downloads) everything, instead of picking up
   only what's still missing — expensive and slow on a flaky connection, which
   is exactly when you need a bootstrap to be resumable.

### Fix: install serially, check before installing, retry per item

```bash
_formula_installed() { brew list --formula "$1" >/dev/null 2>&1; }
_cask_installed()    { brew list --cask "$1" >/dev/null 2>&1; }

_retry_install() {
  local label="$1"; shift
  local attempt=1
  while [ "$attempt" -le "$RETRIES" ]; do
    "$@" && { echo "✓ $label"; return 0; }
    attempt=$((attempt + 1))
  done
  echo "! $label failed after $RETRIES attempts" >&2
  return 1
}

# one entry at a time, skip what's already there, retry only the failure
_formula_installed "$name" || _retry_install "$name" brew install "$name"
```

This turns a single 40-package batch job into 40 independent, checkable,
resumable steps. Re-running the whole bootstrap after fixing one blocker
(signed into the App Store, approved a `sudo` prompt) only touches what's
still missing.

## `set -euo pipefail` + one unguarded external command = the whole script dies

A bootstrap script under strict mode is only as robust as its weakest
external dependency. If step 2 of 7 (installing packages) is a single
unguarded `brew bundle` call and it returns non-zero, `set -e` aborts the
*entire* script — including the steps that matter most for portability
(creating symlinks, wiring up config) which hadn't even run yet.

**Decide explicitly, per step, whether its failure should be fatal.** For a
bootstrap whose job is "get as much of the environment working as possible,"
a package-manager failure usually should not be fatal — surface it loudly,
record that it happened, and keep going:

```bash
step_packages() {
  if ! install_all_packages; then
    warn "some packages failed — see output above; re-run this step later"
    HAD_FAILURES=1   # read by the caller; never let it stop the script
  fi
  return 0            # explicit: a truthy/falsy return here is load-bearing
}
```

The symlink step, the config-wiring step, and the cleanup step are cheap,
local, and idempotent — they should never be blocked by a remote package
manager having a bad day.

## Never hardcode the clone path inside the dotfiles themselves

The natural first version of `.zshrc` sets:

```zsh
export DOTFILES="$HOME/Documents/Workspace/Personal/dotfiles"
```

That's true on the machine that authored it. It is not guaranteed on any
other machine — a different folder structure, a second checkout for testing,
a user who clones somewhere else entirely. The file silently points at the
wrong place, or nowhere.

**Fix: resolve the repo root from the symlink's own on-disk location**,
instead of remembering where you expect it to be. In zsh:

```zsh
# ${(%):-%N} = the path of the currently-sourcing script (this exact file).
# :A          = resolve to an absolute, symlink-free path.
# :h:h        = walk up two parent directories.
# ~/.zshrc -> <repo>/zsh/.zshrc, so two levels up is <repo>.
export DOTFILES="${${(%):-%N}:A:h:h}"
```

Now `.zshrc` finds its own repo wherever it was actually cloned. The general
version of this lesson: **a config file that needs to know "where am I"
should derive that from its own resolved path, not from a value baked in when
it was written.**

## SSH aliases and key filenames are not portable across a from-scratch restore

A dotfiles repo naturally captures the *exact* SSH config from the machine it
was built on — alias names, and the literal key filenames those aliases
point at. That's fine as long as key material is copied byte-for-byte onto
the next machine. It breaks the moment it isn't:

- Restoring keys from a password manager's SSH agent, regenerating a key
  pair, or simply choosing different naming on the new machine, can all
  produce a *different filename* for what is conceptually "the same identity."
- A script, doc, or Brewfile-style config that hardcodes the filename (not
  just the alias) breaks silently — `git@github-personal:...` still resolves
  fine as a remote URL string, but the underlying `IdentityFile` line in
  `~/.ssh/config` may now point at a file that doesn't exist under that name
  on the new machine.

**Treat the alias as the stable contract**, and verify the actual mapping
with `ssh -T git@<alias>` after any restore — don't assume the key filename
survived the trip. If the naming convention changes on a new machine
(`github-personal` → `github-csdev`, `id_ed25519_github_x` →
`id_x_github`), every doc and script that references the alias by name needs
a pass, not just the SSH config file itself.

## A tool's own installer can silently write into your versioned dotfiles

Once `~/.zshrc` is a symlink into a git repo, any CLI that "helpfully"
self-manages its own PATH entry — a language version manager, a cloud SDK
installer, an AI coding tool's CLI — writes *directly into the versioned
file*, not just into a throwaway copy in `$HOME`:

```zsh
# >>> Codex installer >>>
export PATH="$HOME/.local/bin:$PATH"
# <<< Codex installer <<<
```

This is usually exactly what you want (the tool's setup is now captured,
same as everything else). The sharp edge is surprise: an unreviewed
`git diff` in a dotfiles repo can contain changes you never typed, made by a
tool you ran once. Expect it from anything with an "automatically add this to
your shell config" installer step (nvm, pyenv, rustup, gcloud, and most
agentic CLIs), and read the diff before committing rather than assuming every
line came from you.

## A third-party package source can block an unattended install

A formula pulled from a non-official tap (e.g. a vendor's own Homebrew tap)
can require an explicit one-time trust grant before a package manager will
touch it in scripted/non-interactive mode — a step that has nothing to do
with the package itself and everything to do with when the trust grant was
last given on *this* machine. Two separate lessons fall out of this:

1. **Audit, don't just re-export, "whatever the old machine had installed."**
   A machine accumulates entries whose origin nobody remembers; capturing them
   verbatim into a portable bootstrap just relocates the confusion to a
   second machine, at a worse time (mid-bootstrap, not exploratory browsing).
2. **A vendor-specific trust/consent step is a bootstrap dependency like any
   other** — decide up front whether it's automatable (it sometimes is, via a
   one-line `trust`/`--force` flag) or belongs on the manual checklist next to
   "sign in to the App Store."

## The meta-lesson: dry-run and unit tests can't find any of this

Every failure above passed code review, passed a full test suite running
against isolated temp directories, and passed a `--dry-run` pass against the
real machine. None of that exercises: an interactive `sudo` prompt with no
TTY, a real network connection's failure modes, a vendor's trust-store state,
or what a *different* machine's SSH setup actually looks like after a human
restored it by hand. **The first real run, on the actual second machine, is
not a formality — budget for it to surface fixes, and plan for a "clone,
find issues, push fixes, pull on the first machine" loop as part of the
rollout, not as a sign something was designed wrong.**
