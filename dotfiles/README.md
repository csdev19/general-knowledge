# Dotfiles & Machine Bootstrap

Product-agnostic notes on turning a personal workstation into a versioned,
one-command-reproducible repo — the practical failures that only show up
when the bootstrap script runs on real, second hardware.

## Contents

| Doc | Summary |
| --- | --- |
| [bootstrap-lessons.md](./bootstrap-lessons.md) | Why bulk package-manager installs (`brew bundle` and equivalents) are the wrong shape for a bootstrap script, how `set -e` turns one bad package into a script that never reaches its most important step, self-locating config paths (never hardcode the clone location), SSH aliases vs. key filenames surviving a restore, tools that self-modify your versioned rc files, and why dry-run/unit tests can't catch any of this. |
