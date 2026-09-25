# AGENTS.md

Supplemental CLI tools for a [mergerfs](https://github.com/trapexit/mergerfs) pool. Each `src/mergerfs.*` is a standalone stdlib-only Python 3 script; `src/mergerfs.mktrash` is the only other Bash script; `src/mergerfs-tools` is a Bash wrapper that dispatches to them. No build system, test suite, CI, or package manifest.

## mergerfs-tools wrapper

`src/mergerfs-tools` is a thin dispatcher (commands `ctl|fsck|dup|dedup|balance|consolidate|trash|vacate`) that owns a brand-new long-option vocabulary and translates internally to the sibling scripts. It only translates args and `exec`s the sibling script — it contains no tool logic, and the user never types an underlying script's original flags. When editing it:
- The wrapper's option namespace is the contract: common `-n/--dry-run`, `-v/--verbose`, `-h/--help`, `--include`, `--exclude`; command-specific long options (`--copies`, `--keep`, `--ignore`, `--free-gap`, `--mount`, `--fix`, `--check-size`, `--dir-max-files`, `--dir-max-size`, `--prune`, `--strict`, `--min-size`, `--max-size`). Add or rename here, then map internally; never re-expose an underlying letter.
- `--include/--exclude` are pre-validated before dispatch (`build_filters`): literal values (no glob metacharacter) are probed under the target directory and routed as file vs directory, and both missing paths and combinations the underlying script cannot honour (dup/dedup with a `/` path, dup given a directory, consolidate given a file, balance asked to exclude a directory) hard-error. Glob values cannot be probed and are fanned out to every matcher the command supports (basename, full path, directory prune).
- Command-specific values (`--keep`, `--ignore`) are validated and mapped to the underlying vocabulary (e.g. `most-space`→`mostfreespace`, `different-hash`→`diff-hash`); unknown options hard-error rather than passing through.
- Data-changing commands execute by default; `--dry-run` previews. `--dry-run` is impossible for `balance` (no execute switch) and `ctl`; the wrapper rejects it rather than ignoring. `fsck` audits unless `--fix` is given.
- `vacate` only copies, never deletes source: it removes the branch from the pool, then `rsync --ignore-existing` into the pool so nothing there is overwritten.
- It locates siblings via `BASH_SOURCE[0]`, so it only works installed next to, or run from, `src/`. Must pass `bash -n` and `shellcheck -S warning`.

## Cannot run or test on this machine

- Every xattr-based tool (`fsck`, `balance`, `consolidate`, `dedup`, `dup`) does `ctypes.CDLL("libc.so.6")` at import; `ctl` reads Linux `/proc`. All require a live mergerfs mount exposing `user.mergerfs.*` xattrs. They fail immediately on the macOS dev host.
- There is no test harness. Verify syntax only:
  - Python: `python3 -m py_compile src/mergerfs.<tool>` (parses, does not execute — works on macOS).
  - Bash: `bash -n src/mergerfs.mktrash` and `shellcheck -S warning src/mergerfs.mktrash`.
  - Do not run `py_compile` on `mergerfs.mktrash`; it is shell and errors.
- Functional testing needs a Linux box with a real mergerfs mount, root, and `rsync`.

## Install / add tools

- `make install` copies every name in `APPS` to `$(DESTDIR)$(BINDIR)` (default `/usr/local/bin`) with mode 0755. No compilation step.
- To add a tool: drop the script in `src/` and add its name to `APPS` in `Makefile`.

## No shared module — helpers are copy-pasted

`lgetxattr`, `ismergerfs`, `mergerfs_control_file`, `mergerfs_srcmounts`, `match`, `human_to_bytes`, `buildargparser` bodies are duplicated across scripts by design (installed scripts must be self-contained). A fix to shared behavior must be applied to every affected file, not one.

## xattr keys differ per tool

- `user.mergerfs.branches` — written by `mergerfs.ctl add/remove` (renamed from `srcmounts` in commit `3ebaf9e`).
- `user.mergerfs.srcmounts` — still read by `ctl info`, `balance`, `consolidate`, and `mktrash`. Do not blanket-rename across tools; mergerfs versions differ.
- Per-file keys: `user.mergerfs.fullpath`, `basepath`, `relpath`, `allpaths`. Control file is `<mount>/.mergerfs`.

## Destructive defaults

- `mergerfs.balance` runs `rsync` with `--remove-source-files` immediately; there is **no** dry-run flag and it does not default to printing.
- `dup`, `dedup`, `consolidate` only print commands unless `-e/--execute`; `fsck` only audits unless `-f/--fix`.
- `balance`, `dup`, `consolidate`, `mktrash` expect root and `rsync` installed.

## README drift

README lags the code; trust `buildargparser()`/`print_help()` in `src/`. Examples:
- `dedup -i` choices are `same-size|diff-size|same-time|diff-time|same-hash|diff-hash|same-short-hash|diff-short-hash` (README says `different-*`), and `-d` defaults to `mergerfs`, not `newest`.
- `dedup -D/--exclude-dir` exists in code but not in README.

## Style

Match surrounding code: 4-space indent, stdlib only, no type hints, no space after commas in call/def args, old `%` formatting, `main()` guarded by `if __name__ == "__main__":`. Several tools re-wrap `sys.stdout`/`sys.stderr` as UTF-8 `backslashreplace` at the top of `main()` — preserve that when editing.
