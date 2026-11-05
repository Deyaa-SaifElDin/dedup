# dedup
An entry-aware, atomic deduplication Python executable to remove .bash_history duplicate command records!

## What it does

Bash history accumulates duplicates: commands recur, and with extended history each occurrence carries its own timestamp line. `dedup` removes duplicate entries from `~/.bash_history` — or any history file — while preserving chronology, multiline commands, byte fidelity, and the file's exact format. Rewrites are atomic, backups rotate automatically, and the default run is a dry run.

`HISTCONTROL=erasedups` exists, but it deduplicates inside each live shell session: concurrent sessions hold separate in-memory copies, and per-session line counters desynchronize, so re-imported lines get written back as duplicates on the next save. `dedup` is an offline file filter instead: it operates on the file itself, once, with no live-session bookkeeping. Run it, then re-read the file in open shells (`history -c; history -r`).

## Design methodology

The tool was designed by explicit problem classification before any implementation.

### Problem class

Streaming, order-preserving, exact duplicate elimination. In the standard taxonomy this is a hashing problem: hash-set membership over a canonical key, one deterministic pass. It is deliberately not:

- Sorting. `sort -u` removes non-adjacent duplicates but destroys chronology — the file's only ordering metadata is position — and restoring order costs an index reconstruction on top of O(n log n). Membership testing is O(n) with order kept for free. (GNU coreutils documents the complementary split: `uniq` detects only adjacent repeats and points to `sort -u` for the non-adjacent case; [uniq](https://www.gnu.org/software/coreutils/manual/html_node/uniq-invocation.html), [sort](https://www.gnu.org/software/coreutils/manual/html_node/sort-invocation.html).)
- Compression. Deduplication selects entries; compression re-encodes every byte. The output of `dedup` is a byte-subset of its input, readable by bash with no decoder.

### Layer separation

Four concerns, each independently testable:

| Layer | Responsibility | Implementation |
|---|---|---|
| Segmentation | split the byte stream into entries (timestamp marker + command block) | 2-state streaming scanner, bytes regex `^#1[0-9]{9}$` |
| Key policy | decide what counts as a duplicate | canonical key, specified below |
| Membership | detect duplicates | built-in `set` of bytes keys |
| Persistence | backup, atomic rewrite, newline state | `mkstemp` + `os.replace`, hardlink backup |

The key policy is specified first and frozen; mechanism follows. Markerless files (no timestamp lines) are handled as one entry per line.

### Key policy

Default: a duplicate is the same command text, regardless of timestamp. The key is the command block with each line rstripped of ASCII whitespace; the timestamp marker is excluded. Trailing-whitespace variants of one command collapse into a single entry — the same normalization atuin applies on import: "trailing whitespace means nothing, breaks dedupe, and wastes a few bytes" ([atuin PR #3390](https://github.com/atuinsh/atuin/pull/3390)).

Orphan markers (a timestamp with no command) and unmarked leading content are keyed under NUL-prefixed sentinels. NUL cannot occur in a bash history line (readline C strings), so sentinel keys cannot collide with command text; both are always preserved.

Tie-breaks: keep the first occurrence (default; chronology-stable) or the last (`--keep-last`). `--exact` switches the policy to byte-identical entries only (timestamp and block must both match).

### Data types

- Bytes end to end. The file is opened in binary mode, keys are `bytes`, output is `bytes`. No decode/encode round trip, and none of the `str.splitlines()` pitfalls: that method splits on `\r`, `\v`, `\f`, `\x1c`–`\x1e`, `\x85`, and U+2028/U+2029; here only `\n` terminates a line, by construction.
- `set` for membership. CPython's set is an open-addressing hash table whose own header states it exists "primarily about membership testing" ([Objects/setobject.c](https://github.com/python/cpython/blob/main/Objects/setobject.c)); keys are compared by value — exact, so no digest-collision analysis is needed.
- `dict` for `--keep-last`: overwriting a key keeps the last index per key; a second pass emits at the recorded positions.
- Memory is O(u) — u = total unique canonical bytes — plus one entry buffer, not the file size.

### Multiline commands

With `cmdhist` and `lithist` enabled, bash stores multiline commands as consecutive lines under one timestamp. `dedup` keys and compares whole blocks, so multiline duplicates collapse as units and are never split or merged line-wise.

### Atomicity and backups

- Rewrite: kept entries stream into a `tempfile.mkstemp` file in the same directory, then `os.replace` performs an atomic rename — readers never observe a partial or missing file ([rename(2)](https://www.man7.org/linux/man-pages/man2/rename.2.html)).
- Backup: `os.link` hardlink — O(1), same directory guarantees the same filesystem ([link(2)](https://www.man7.org/linux/man-pages/man2/link.2.html)) — with a streaming-copy fallback, rotated as `.bak`, `.bak.1` … `.bak.N`.
- The original trailing-newline state is preserved. If nothing would be removed, the file is not touched.

## Usage

Requirements: Python 3.8 or later, standard library only (no third-party dependencies).

```console
$ ./dedup                       # dry run: report only, no writes
file: /home/user/.bash_history
mode: command text, keep-first
entries: 865  kept: 288  removed: 577  multiline: 0
(dry run: no file modified; pass --apply to rewrite)

$ ./dedup --apply               # rewrite the file, rotate backups
$ ./dedup --apply --quiet       # no report (cron / scripted use)
$ ./dedup --keep-last --apply   # keep the LAST occurrence instead
$ ./dedup --exact --apply       # only byte-identical entries
$ ./dedup --history /path/to/file --apply
```

| Option | Effect |
|---|---|
| (none) | dry run: report entries kept/removed, no writes |
| `--apply` | rewrite the file |
| `--quiet` | suppress the report |
| `--keep-last` | keep the last occurrence per command text (default: first) |
| `--exact` | remove only byte-identical entries (timestamp + block) |
| `--collapse-commands` | deprecated alias for `--keep-last` |
| `--no-backup` | rewrite without writing a backup |
| `--backup-generations N` | retain N rotated backups (default 8; 0 = a single `.bak`) |
| `--history FILE` | target file (default `~/.bash_history`) |

Exit codes: 0 = success (including dry run, or nothing to remove), 1 = file not found, 2 = incompatible options.

Concurrent sessions: run `--apply` when no other interactive bash session is open, or re-read the file afterwards in each live shell:

```bash
history -c; history -r
```

Periodic automation example (cron, every six hours at quiet hours):

```cron
0 */6 * * * /usr/local/bin/dedup --apply --quiet
```

## Complexity

Time O(n) — one pass over the lines, or two passes for `--keep-last`. Memory O(u) — the unique-key set plus one entry buffer. The hash-set design holds until the unique-key set approaches RAM; the documented fallbacks for that regime are external sorting (`sort -u`) or a SQLite staging pass — the design atuin and mcfly use for persistent history stores. For a bash history file, that boundary is not approached in practice.

## References

- [GNU Bash manual](https://www.gnu.org/software/bash/manual/html_node/Bash-Variables.html) — `HISTCONTROL`
- [GNU coreutils](https://www.gnu.org/software/coreutils/manual/html_node/uniq-invocation.html) — `uniq`, `sort`
- [atuin](https://github.com/atuinsh/atuin/pull/3390) — import canonicalization and index deduplication ([#1610](https://github.com/atuinsh/atuin/pull/1610), [#4347](https://github.com/atuinsh/atuin/pull/4347))
- [CPython](https://github.com/python/cpython/blob/main/Objects/setobject.c) — set implementation; [tutorial](https://docs.python.org/3/tutorial/datastructures.html#sets)
- [tempfile](https://docs.python.org/3/library/tempfile.html#tempfile.mkstemp), [rename(2)](https://www.man7.org/linux/man-pages/man2/rename.2.html), [link(2)](https://www.man7.org/linux/man-pages/man2/link.2.html)
