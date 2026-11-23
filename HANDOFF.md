# HANDOFF — picking up #103

This file is a session bridge, not a durable document. It carries only
what is needed to resume one unfinished thread. The Human's standing
conventions live in README's *Contributing*; settled decisions live in
`DECISIONS.md`; open work lives in the
[issues](https://github.com/andy-emerson/TallyDB/issues). Nothing here
restates those — if this file disagrees with them, they win.

Note on its existence: `HANDOFF.md` was retired in `074aeac` on the
grounds that its contents had homes elsewhere. That reasoning still
holds for conventions and decisions, and this file deliberately does
not carry either. It exists because one research thread stalled
mid-conversation and its coordinates went stale underneath it. Fold it
into #103 and delete it once that issue is ruled.

## State

`main` is `ad808df` (2026-09-13). The working branch
`claude/artifact-session-i65b7v` carries only this file. Note that the
repository convention is to develop on `claude/dev`; this branch name
came from the session harness, so the Human may want this moved before
it lands.

Nothing is awaiting review or merge, and no code work is in flight.

## The one live thread: #103

[#103](https://github.com/andy-emerson/TallyDB/issues/103) — *the
Python binding should target Python, not NumPy*. Research is complete
and recorded in two comments on the issue. **No ruling has been made.**
Nothing may be built until one is, because every live option freezes a
public interface.

The issue is labelled `enhancement`. It carries unruled decisions, so
by the README's own convention it should carry `decision`.

### What the research settled

Two of the six decisions originally filed are closed, both because the
Human caught them:

- **The data-out contract is not an open decision.** The Arrow
  PyCapsule interface (`__arrow_c_stream__` plus `__arrow_c_schema__`)
  is the only protocol anything consumes, and NumPy-as-the-contract is
  not a trade-off to weigh — it is refused by two standing rulings. See
  *NULL is placed, not ordered* (`DECISIONS.md:1750`) and *NULL crosses
  to Lua as a sentinel* (`DECISIONS.md:1464`), whose stated principle
  requires every boundary value mapping to be a total, invertible
  function. An f64 array cannot hold NULL distinctly from NaN, so it
  cannot be one. That ruling is written for the Lua boundary; applying
  it to the Python boundary is the agent's reading, not a ruling the
  Human has made.
- **Whether NumPy is a declared wheel dependency is not a decision
  either.** The binding is an API *for* NumPy, not a dependant of it; we
  declare nothing and the user brings their own. The real question
  underneath it is what a kernel receives, below — if the kernel seam
  hands out objects only NumPy can read, NumPy becomes a requirement in
  fact without ever being declared.

### The four decisions still open

1. **What a host-callback kernel receives.** The one that matters; the
   other three lean on it. Five options are laid out on the issue,
   spanning per-window versus whole-run granularity and four container
   types. Measured: per-window is **118× slower** than handing the
   kernel one strided view over the whole run.
2. **Binding tooling** — PyO3 and maturin, cffi, raw ctypes, or a
   hand-rolled C extension; plus an ABI sub-fork. Constrained by (1):
   raw ctypes cannot implement the buffer exporter safely, and `abi3`
   buffer-protocol support starts at Python 3.11.
3. **Whether a result is a re-playable handle or a one-shot cursor.**
   Freezes with the data-out object model.
4. **The threading model** — one `Mutex`-wrapped handle shared across
   threads, an unsendable single-threaded handle, or first-class reader
   and snapshot objects.

### Coordinates, corrected

**Every file path and line number in the #103 comments is wrong.** They
were written before the `crates/*` → `src/*` restructure. The ones that
matter, re-verified against `ad808df`:

| What | Where now | Was |
|---|---|---|
| `trait LogWriter: Send` | `src/storage_lite/io.rs:91` | `crates/storage-lite/src/io.rs:89` |
| `WindowAggregate` | `src/query_lite/exec.rs:68` | `crates/query-lite/src/exec.rs:66` |
| `Send + Sync` assertion | `src/table.rs:2919` | `crates/engine/src/table.rs:2878` |
| "host-callback NumPy kernels" | `DECISIONS.md:1202` | `DESIGN.md:541` |
| NULL/NaN ruling | `DECISIONS.md:1750` | `DESIGN.md:250` |
| Sentinel ruling | `DECISIONS.md:1464` | `DESIGN.md:2147` |
| M5.5 wheels-on-PyPI ruling | `DECISIONS.md:946` | `DESIGN.md:1202` |

### Facts re-verified against `ad808df`

- `trait LogWriter` still carries `Send` and not `Sync`, so `Table`,
  `Database` and `MaterializedView` are `Send` but **not** `Sync`. A
  `Mutex` restores it, and that is the same wrapper the single-writer
  model needs anyway.
- `WindowAggregate` is still `Send + Sync` over `&[&[f64]]`, with no
  array library in any signature.
- `DECISIONS.md:1202` still reads "host-callback **NumPy** kernels",
  which presumes the answer to decision (1). It needs a doc pass once
  that is ruled, whichever way it goes.

Measured earlier in the research and not re-run since: the 118× figure,
DuckDB calling `__arrow_c_stream__` three times per query, and one of
our exported streams returning 0 rows silently on reuse rather than
failing. Treat these as cited to their original run, not as current.

## Open follow-up, independent of the ruling

`src/table.rs:2919` asserts `Send + Sync` for `TableReader` and
`TableSnapshot` only. A binding will rely on `Table` and `Database`
being `Send`, and on their *not* being `Sync`, and nothing holds either
fact today. Worth an assertion regardless of how #103 is ruled.
