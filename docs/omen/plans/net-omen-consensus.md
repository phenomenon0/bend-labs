# net × omen — one line again, and what each side knows (2026-09-24)

There are two omen lines. `claude/next-gen-tier-review-xss592` ("net") re-ported the
omen stack on 2026-09-17 in commits of its own, then built the networking library on
top, and took upstream up to 2.0.25+2. `omen` went on to 2.0.26. Their histories share
nothing since `ff7a40c`: 274 net-only commits, 17 omen-only, and none patch-equivalent.
Their content has converged anyway (identical `base.bend` core, `bend.ts` and lane
docs), so a merge conflicts in **3 files and 12 hunks**.

## What net has that omen needs

| what | why it matters to omen | evidence |
|---|---|---|
| **X64 layout** (`dcdcf13`): U64, I64 and F64 ride boxed wherever C drops a Term | **omen has a live memory fault.** A full 64-bit word passed through a polymorphic parameter, a generic field or a union slot is freed as a pointer | `tests/base/x64_boxed.bend` on omen `1e919d1`: prints `ok`, then `bend: memory fault`. It passes on the merge |
| **native `Bytes()`** (47 defs over the packed-string runtime) with **borrowed `peek` natives** | a scan loop touches no reference count. It is Data, so it forks without `Array.fork` | `tests/base/bytes_*` |
| **`File.read_buf(file, max) -> Bytes()`** | the file-intake bottleneck the monoids lane measured (0.5 s for 55 MB through List chunks) | `monoids.md`, round 3 |
| **epoll IO loop** (Linux; select stays on macOS) | the scheduler fix that also exists as `fix/epoll-*` and in two other branches: one copy, not four | `bench/net` |
| **`--export`** core terms, for the kernel | a second checker | `kernel/` |
| `power/deflate`, `gzip`, `inflate_proof`, `json_value` | proven codecs for power/ | their laws and proofs |
| `net/` (HTTP, TLS, WebSocket, streaming bodies, router), `wire/` | the library itself, with LAWS/PROOF, mutants, and outside checks | `net/README.md` |

## What omen has that net needs

| what | why |
|---|---|
| **upstream 2.0.26** (named packages, comp.ts's shared match emitters, the channel rows) | net is one release behind, and upstream ships daily |
| **the F64 literal match** on the C lane (`ec59bbe`, `tests/f64/match_literal.bend`) | the 64-bit match path took U64 and I64 only |
| `power/exact` (Kulisch sum) and `power/utf8` (DFA-map validator, `first_bad`) | exact reductions, and UTF-8 checking that a server's reader could call |
| `demos/monoids` | the split-invariance discipline: every program recomputes at several depths and prints `same` |
| the vm and translator differential fuzzers | |

## Where both are wrong: size

Neither line passes `gates/repo.ts`. The merge has the same ten reds as net's own tip,
and adds none:

| file | cap | net tip | omen | merge |
|---|---:|---:|---:|---:|
| `base.bend` | 48,928 / 49,777 | 62,708 | 49,777 | 62,700 |
| `comp.ts` | 85,100 / 84,100 | 95,952 | ~83,000 | 93,262 |
| `effs/tls_listen.c`, `io_http_engine/*`, `deflate_proof`, `deflate.dat`, `NETWORKING.md` | per rule | over | — | over |

This runs against `FLOW.md` ("caps are budgets, not thermometers") and against
`plans/upstream-offer.md` (the fork's cost is its merge). Most of it is where it
belongs, in Zone B (`net/`, `wire/`, `demos/`). The part that is not is **+13k
`base.bend` (Bytes, TCP, TLS, DNS) and +10k `comp.ts`**. The fix is the regex plan
applied to Bytes: a module over natives reachable from outside Base, not more Base.

## The merge (bend `5b3845c`, `30156ee`, `a0ccfe0` on `claude/bend-primitives-deep-dive-b9g079`)

`omen` plus `net`, resolved as follows:

- `comp.ts`:
  - **omen's (2.0.26) structure** where the two collided (`emit_match`, the OPTIMIZED
    rows);
  - **net's** X64 layout, `lay_pack` boxing, and `peek` natives, with omen's compact
    `array_*` rows beside net's `bytes_*` rows;
  - net's epoll loop;
  - one guard for net's widened `JS: Gen` in omen's `emit_row`.
- `main.ts`: both option sets (upstream's publish/link/login, net's `--export`).
- `repo.ts`: omen's caps. Neither side's caps are true today; the size section above is
  the real decision.

- `effs/signal_pending.c`: a conflict git could not see. Upstream 2.0.26 moved
  `chan_bool` into `effs/chan.c`, which only channel programs carry, so
  `net/examples/ws_chat` stopped building. The effect now spells its Bool itself.

### Verification: merged tree vs each parent

| check | merge | net tip | omen |
|---|---|---|---|
| `bend net/PROOF.bend`, `net/ws_proof.bend` | All terms check | same | — |
| `net/mutants.py` | 32/32 killed | same | — |
| `net/check.py` (examples built, checked from outside) | all passed (after the `chan_bool` fix) | all passed | — |
| `demos/monoids/run.sh` | 25/25 | — | 25/25 |
| `tests/power` exact, utf8 | 6/6, 6/6 | — | 6/6, 6/6 |
| f64 | 21/22 (gpu) | — | 21/22 |
| codex | 161/161 | — | 161/161 |
| `tests/base/x64_boxed` [c] | ok | ok | **memory fault** |
| `tests/base/bytes_ops`, `bytes_scan` [c] | red | **same red** | (absent) |
| strings | 99/102 (deep interpret timeout, gpu, `io_sock_utf8` [c] timeout) | `io_sock_utf8` same red | 100/102 |
| repo gate | 71/82 (the size table above) | 71/82 | 56/56 |

The merge adds no red of its own. All of its reds are red on one parent too.

## Knowledge moved in the second commit

The net line rewrote `power/bytes.bend` over the native `Bytes()`. `power/utf8`
therefore moved onto it, and `power/exact` gained `sum_bytes`:

- **Read, from `File.read_buf`:** 0.5 s → 0.1 s for 55 MB. That was the bottleneck
  the monoids lane measured.
- **End to end at 4 threads:**
  - UTF-8: 571 → 217 ms
  - exact sum of 2^24 floats: 564 → 200 ms
- **No unsafe code left in utf8:** a Data buffer forks without `Array.fork`.
- **Per-byte compute is about 1.7x the raw Array cells.** Every `Bytes` read runs
  `str_peek_ro`, which decodes the view, offset and width. That is net's next
  runtime target: a bulk borrowed read, or the descriptor hoisted out of the loop.

## Open for the net line's owner

1. `tests/base/bytes_ops` does not build on the C lane, and `bytes_scan` prints
   wrong values at the end of the buffer (256 / 0). Both are on net's own tip.
2. Size: move `Bytes`/`TCP`/`TLS` out of `base.bend` the way the regex plan moves
   regex, or raise the caps with the itemized surface `FLOW.md` asks for.
3. Take this merge back: `git merge claude/bend-primitives-deep-dive-b9g079` into the
   net branch is a fast merge now. After that, both lines are one.
