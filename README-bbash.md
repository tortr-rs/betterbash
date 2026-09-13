# bbash (Better Bash)

bbash is GNU Bash with a small, additive set of clearer-named commands
built in. It is a straight fork of the real Bash source tree
(git.savannah.gnu.org/cgit/bash.git, GPLv3) — currently based on **Bash
5.3, patch 16**.

## Compatibility guarantee

**Every existing script, command, and piece of syntax behaves exactly as
it does in real Bash.** The parser, expansion rules, and every existing
builtin are unmodified. Nothing in this fork can collide with or change
existing behavior, because everything it adds is a brand-new builtin name
that does not exist in stock Bash.

This has been verified by running real, unmodified scripts under both
stock bash and this build and diffing the output byte-for-byte, including:

- Bash's own 24,853-line `configure` script (stdout and every generated
  file, e.g. `config.h`, `Makefile`)
- `makepkg --help` / `makepkg --printsrcinfo` on a real PKGBUILD
- `pacman-key --help`
- Bash's own test suite (`tests/run-all`) — identical pass/fail results

If you find any behavioral difference from real Bash for something that
isn't one of the new commands below, that's a bug — please report it.

## Building

```sh
./configure
make
./bash        # or ./bbash, a symlink to the same binary
```

No extra configure flags are needed. The only new build-time dependency
is `libarchive` (used by `tarzan`); everything else (the alias builtins,
`seek`) uses only the C library already required to build Bash itself.

## What's new

### Clearer-name aliases

These are new builtins that dispatch straight through to the real,
unmodified external command — same arguments, same exit status, same
redirection/pipeline/job-control behavior as if you'd typed the real
command yourself. The originals are completely untouched and still work
exactly as before.

| bbash command | runs         |
|----------------|--------------|
| `list`         | `ls`         |
| `copy`         | `cp`         |
| `move`         | `mv`         |
| `delete`       | `rm`         |
| `make-dir`     | `mkdir`      |
| `ownership`    | `chown`      |
| `word-count`   | `wc`         |
| `link`         | `ln`         |
| `disk-usage`   | `du`         |
| `disk-free`    | `df`         |
| `processes`    | `ps`         |
| `permissions`  | `chmod`      |
| `stop-process` | `kill`       |

`stop-process` dispatches to the shell's own `kill` builtin (not an
external binary), so job specs like `%1` still work as expected.

```sh
list -la /tmp
copy a.txt b.txt
delete b.txt
```

### `tarzan` — a clearer interface to tar

```sh
tarzan --create backup.tar.gz --quiet ./src
tarzan --extract --quiet backup.tar.gz
tarzan --list backup.tar.gz
```

Reads and writes fully standard tar archives (byte-compatible with real
tar — verified against GNU tar in both directions) via libarchive, with
gzip/bzip2/xz/zstd compression detected transparently: from the archive's
own contents on read, and from the archive name's extension
(`.tar`/`.tar.gz`/`.tgz`/`.tar.bz2`/`.tbz2`/`.tar.xz`/`.txz`/`.tar.zst`/`.tzst`)
on write. No compression flag needed.

| Flag | Short | Meaning |
|---|---|---|
| `--create <archive>` | `-c` | create ARCHIVE from the given files/dirs |
| `--extract` | `-x` | extract the archive named by the trailing argument |
| `--list` | `-t` | list contents without extracting |
| `--output <path>` | `-o` | (extract only) directory to extract into |
| `--quiet` | `-q` | suppress the summary/progress output |
| `--verbose` | `-v` | show per-file operations |
| `--exclude <pattern>` | `-e` | exclude matching paths (repeatable) |
| `--preserve-permissions` | `-p` | also restore original ownership |
| `--follow-symlinks` | `-L` | (create) store symlink targets, not the links |

Exactly one of `--create` / `--extract` / `--list` is required. Flags are
never bundled (`tarzan -xzvf` doesn't exist) — each is separate and
explicit.

### `seek` — a clearer interface to grep

```sh
seek --pattern "TODO" file.txt
seek --pattern "error|warn" --recursive ./src
```

Real POSIX regex matching (extended by default, like `grep -E`; add
`--basic-regex` for traditional BRE syntax) via the C library's own
regex engine — the interface is simpler, the matching power isn't reduced.

| Flag | Short | Meaning |
|---|---|---|
| `--pattern <regex>` | `-p` | the search pattern (required) |
| `--recursive` | `-r` | search a directory tree (default `.` if no path given) |
| `--ignore-case` | `-i` | case-insensitive match |
| `--invert` | `-v` | show non-matching lines |
| `--count` | `-c` | show only a per-file match count |
| `--line-numbers` | `-n` | show line numbers (**on by default**) |
| `--no-line-numbers` | | turn line numbers off |
| `--files-only` | `-l` | list matching filenames only |
| `--context <n>` | `-C` | show N lines of context around each match |
| `--basic-regex` | `-G` | POSIX basic regex instead of extended |
| `--color` / `--no-color` | | force match highlighting on/off |

With no file argument (and `--recursive` not given), `seek` reads standard
input, like grep.

Output adapts to where it's going: when stdout is a terminal, matches are
grouped per file with colored highlighting (line numbers on by default);
when piped or redirected, output is plain `file:line:text` per match — one
self-contained line per match, easy to pipe into `cut`/`awk`/etc. Match
*sets* are identical to real `grep -E` either way — verified against a
real source tree.

Exit status matches grep's convention: `0` if something matched, `1` if
nothing matched, `2` on a bad pattern, bad usage, or an unreadable file.

## Design notes

- The alias builtins and `tarzan`/`seek` are true Bash builtins (linked
  into the `bash`/`bbash` binary), not external wrapper scripts — they
  don't depend on the host having a particular version of `ls`, `tar`, or
  `grep` installed, except that `tarzan`/the aliases still shell out to
  the real underlying tool for their own work where that's the point
  (e.g. `list` really does run `ls`).
- `tarzan` always applies tar's standard extraction safety checks
  (rejects absolute paths, `..`, and symlink tricks in archive entries)
  and there's no flag to turn that off.
- Source layout: the new commands live in `builtins/clearcmds.def`,
  `builtins/tarzan.def`, and `builtins/seek.def`, following the same
  `.def` → generated `.c` convention as every other Bash builtin.
