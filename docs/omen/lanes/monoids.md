# monoids — three sequential loops as fork trees (2026-09-23)

Branch `claude/bend-primitives-deep-dive-b9g079` (bend fork), on top of the 2.0.26 sync
(`sync-2.0.26.md`). New files only: `demos/monoids/**`. Nothing in `bend2/`, `power/`,
`tests/` or `gates/` was touched by this lane.

## The pick, and why

This lane went through the deep dives (`phenomenon0/deep-dives`) looking for what Bend
does better than a C loop, given the power lanes' two measured rules:

1. A fork must beat ~3x before it pays, unless its leaf only does arithmetic. power-3
   measured a ~0 tax on an arithmetic leaf and ~3x on one that allocates.
2. A shared Array does not fork (power-1). A leaf should own nothing.

Rule 1 wants leaves that are pure arithmetic on addressed data. Rule 2 wants that data to be a
function of the index. Rule 1 also needs something from the join: an associative summary
small enough to be kind `Data`. Three deep dives each contain a loop that looks
sequential but has such a summary:

| deep dive | the loop | the summary | size |
|---|---|---|---|
| unicode + parsers | UTF-8 validation (DFA) | state→state map (Mytkowicz et al. 2014) | 1 U32 |
| floating point ("Kahan", "fsum", "-ffast-math breaks associativity") | `sum += x` | Kulisch fixed-point integer | 10 U32 |
| rng ("LCG", "PCG") | `x = a*x + c` | affine map (Brown 1994) | 4 U32 |

Candidates considered and left out: MT19937 jump-ahead (GF(2) polynomial arithmetic
over a degree-19937 modulus: not small); a JSON bracket/quote bicyclic monoid (the
best next one, but its data is not a function of the index, so it needs a file
and the Array fork path); rANS or Huffman compression (the output is a byte stream,
which allocates in the leaf); an IIR filter as an affine scan (in F32 it is not
associative, and in fixed point audio's rounding breaks linearity).

## What shipped

| file | what |
|---|---|
| `utf8.bend` + `utf8_gen.py` | 11 cases (clean, cut mid-scalar, starting mid-scalar, 8 poisons, one per rule of table 3-7) |
| `kulisch.bend` + `kulisch_gen.py` | 3 random cases + 9 hand-built rounding edges |
| `pcg.bend` + `pcg_gen.py` | reference demo outputs, 2 jumps (2^40+7, 10^12), the tree count against the loop and against per-leaf reseeding |
| `*_big.bend`, `kulisch_drift.bend`, `pcg_twin.c` | the scale runs; the C sequential reference |
| `run.sh` | oracle + interpret + js + c (1 thread) + c (4 threads) per program; `BIG=1` times the scale runs |

Each program also checks itself for split invariance at runtime: it recomputes at
several tree depths, and prints `same` or `DIFFER`.

## Battery

`demos/monoids/run.sh`: **15/15** (3 programs x oracle, interpret, js, c-1, c-4), on the
omen line before the sync and again on the synced branch; `gates/repo.ts` 56/56.

## Measured

Quiet 4-vCPU Xeon @ 2.10 GHz, clang -O3, `--gpu off`, best of 3, on the synced branch
(`d6792ca`). Identical output at every thread count.

| run | 1 thread | 2 | 4 | speedup | output |
|---|---:|---:|---:|---:|---|
| `utf8_big`, 2^28 bytes | 2.96 s | 1.41 s | 0.77 s | 3.8x | `map=08888888 cps=159382215` |
| `kulisch_big`, 2^26 F32 | 2.16 s | 1.09 s | 0.63 s | 3.4x | `exact 0xbe9ac177` (naive: `0x3bdda8fe` / `0x3b0c3ca8`) |
| `pcg_big`, 2^31 draws | 6.05 s | 2.96 s | 1.61 s | 3.8x | `hits 843353130` = `pcg_twin.c` (2.91 s sequential) |

Commit: `0c67c15` (bend fork, `claude/bend-primitives-deep-dive-b9g079`).

## Deviations and findings

1. **The split self-check caught my own bug.** The first convergence-optimized `utf8` leaf
   assumed convergence even when the leaf was shorter than 4 bytes. Depths 11 and 15
   (1-byte leaves) printed `DIFFER` while the verdicts still matched CPython. The fix was a
   leaf under 4 bytes keeping its full map. Without the in-program depth comparison, this
   would have shipped green, because the oracle only sees the depth-7 answer.
2. **A branch chain in the byte classifier cost 1.7 s of 4.9 s.** The emitted C is a
   data-dependent branch per byte, and random mixed-script text mispredicts it. It was replaced by a
   telescoping sum of `[b >= edge] * delta` over the 13 edges (4.9 s → 3.2 s on 2^28 bytes).
3. **Unrolling the leaf to one Threefry slot per turn bought almost nothing** (3.20 →
   3.06 s). The remaining cost is the DFA's latency chain plus synthesizing the corpus, not
   Bend's per-iteration overhead. The unrolled form stays because it halves the slot
   lookups.
4. **A local named like a def breaks the importing file.** `+mant = …` inside
   `kulisch.bend` checks on its own, but fails when the file is imported (`import
   ./kulisch.bend as K`) with "expected a quantified datatype after +", because `mant` is
   also a def there. The local was renamed to `sig`. This looks like an upstream parser bug
   worth filing: the same file should not check or fail depending on how it is loaded.
5. **The first kulisch draw correlated sign with exponent.** Both came from bit 31 of one word,
   which biased the sea's mean (-95 over 2^26 draws). This was fixed in the Bend file and the
   oracle together, before any number was recorded.
6. **No U32↔U64 conversion exists on omen**, so `pcg.bend` does its 64-bit arithmetic on U32
   pairs (a `mulhi` from four 16-bit products). That makes it run on upstream Bend as-is,
   and it runs at 2.1x the sequential C twin on one thread (and beats it on four).

## Round 2 — lostupdate and probe (bend `3759f41`)

These are the top two picks from `plans/monoids-next.md`. `run.sh` passes **25/25**
(5 programs x 5 lanes), and `gates/repo.ts` passes 56/56.

| run | 1 thread | 2 | 4 | speedup | output |
|---|---:|---:|---:|---:|---|
| `lostupdate_big`, all 601,080,390 schedules (n = 8) | 32.88 s | 16.66 s | 8.14 s | 4.0x | `min=2 x8 correct=12870`, full histogram = DP oracle |
| `probe_big`, 2^24 slots x loads 50/75/85/95 % | 2.21 s | 1.20 s | 0.64 s | 3.5x | = CPython insertion at 50 % and 85 % |

- **lostupdate refutes the concurrency deep dive's "or all of them".** The final count
  is never below 2. Exactly 8 schedules reach 2, for every n from 3 to 8, and exactly
  C(2n, n) schedules are correct. The enumeration is brute force by design: the
  oracle's DP answers n = 8 in 45 ms. What Bend shows is a 601M-way search whose
  histogram no split can change.
- **probe: a real hash on a real table does not match Knuth.**
  - At 2^24 slots, every load comes out below the uniform-hashing means. At 95 % load,
    the miss cost is 178.47 against Knuth's 200.50, and the hit cost 9.69 against 10.50.
  - At 2^16 slots and 85 % it is above them (hit 4.14 against 3.83). With sequential
    keys through a two-multiply mixer, finite-size and hash-structure effects go both
    ways.
  - The (max,+) carry map and the ring's fixed point (Q = n − M < 0, so the inflow is
    P) are the whole trick. Pass 2 reuses pass 1's tree of maps as its fork tree.
- **Limits:**
  - probe's sums are U32. At 2^24 slots and 95 % load, the miss total is 2.98e9, so
    loads past about 96 % need a wider counter.
  - lostupdate's schedule word is 32 bits, which caps it at n = 8.
- **Bend notes:**
  - `1n++q` is a pattern, not an expression. Use `U32.inc(U32.from_nat(q))`.
  - A parallel-let's results are affine. To reuse them, pass them to a def with `+`
    parameters (`node(+l, +r)`).
  - A second `match` on another parameter inside a case is refused, because
    scrutinees go in binder order. So `walk` matches only the tree, and the tree's
    shape is the split.
