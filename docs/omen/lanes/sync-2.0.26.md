# Sync 2.0.26 — upstream's 63 commits onto the omen line (2026-09-23)

One merge, no lanes. `up/main` = `6a77e12` ("The flake names 2.0.26") merged onto `omen` = `81f12fc`,
merge base `6018e28`. Committed as `d6792ca` on `claude/bend-primitives-deep-dive-b9g079`; not pushed.

| step | commit | |
|---|---|---|
| the merge | `d6792ca` | `sync 2.0.26` — upstream `6a77e12` onto `81f12fc` |

Upstream's span is 2.0.19 → 2.0.26: the effects consolidated (`chan.c`, `audio.c`, one reader and one
writer per file effect), literals as one `Lit` node in the checker, `Array.map` as a get/set loop,
`@unsafe` array forks, the word match comparing the whole word, one `BEND_RTC` macro, one native
descriptor per constructor in the JS emitter, the shared match tables and arms ("comp.ts sheds
2.8k tokens"), and names on the hub (`--publish name@version`, `bend link`, `bend login`).

## The merge: where the two lines disagreed

Two files conflicted: `bend2/comp.ts` (11 hunks), `gates/repo.ts` (2 hunks). Everything else merged
clean, `bend2/bend.ts` included (taken as upstream's plus our F64 `d` literal, not edited).

Taken from upstream, whole: the null-prototype `WORDS`; the per-constructor native descriptor
(`{T: {C: {intr, elim, cond}}}`); the compacted `array_*` rows (`array_rmw`, `arr_op`); `adt_of`
(`mat_adt` gone) and the match emitter's `emit_nats` / `emit_lits` / `lits_rows` / `emit_tab`
structure; the empty word as a literal `0` in `emit_ctr`; `BEND_RTC`; the compact heap-free class
computation; the comment fix in `emit_intr`; the tests row's optional subdirectory; the bend.ts cap.

Kept ours, each divergence named:

1. **`WORDS` keeps F64, U64 and I64 as W64**, inside upstream's null-prototype form.
2. **The `string_*` / `regex_*` / `map_bit` native rows stay ours**, above upstream's compacted
   `array_*` rows.
3. **`Char`'s elim stays `char_code($0)`** (it reads a `RawChar` too) and **`SCon`'s intr stays
   `str_prepend(h, t)`** (it refuses a non-scalar `Char` in a JS string), both moved into upstream's
   descriptor shape. Upstream's `$0.codePointAt(0)` and `(h + t)` would silently accept what the
   strings design refuses.
4. **`emit_intr` keeps the `facts_hot` block** for the native aggregate builders.
5. **`term_word` keeps its width argument** (`term_word(e, w, 32|64)`): upstream dropped it with the
   W32-only runtime; ours reads 64-bit words.
6. **The Metal-no-fp64 comment** rides with upstream's `BEND_RTC` guard.
7. **The heap-free path keeps `TAG_STR`** (one packed cell, class 1) inside upstream's compact form.
8. **C `io_str` and JS `io_text` stay ours**, as at 2.0.17 (deviations 3 and 4 there): the adaptive
   1/2/4-byte decoder and the fatal-probe + per-byte replacement, pinned on both hosts by
   `tests/strings` and `tests/regex`.

## Semantic conflicts outside the markers

- **The F64 / U64 / I64 native rows** sat outside the conflict in the old
  `{intr: {F64: ..}, elim: {F64: [..]}}` shape. Under upstream's descriptor lookup
  (`native[k].intr`) they would have been read as constructors named `intr` and `elim`. Converted to
  `{F64: {intr}}` etc., like upstream's `U32` / `F32`, whose elim moved into the match emitter.
- **The word-literal matcher was W32-only.** Upstream's `emit_lits` returns `null` unless the word
  is W32; `lits_cond` writes JS numbers; the C match holds the scrutinee at `W32`; the JS match reads
  `u32_to_word(f32_bits(s))`. Our 64-bit words would have fallen through to the constructor path and
  died there. Now:
  - `lits_bits(adt)` reads the width from `WORDS` (32 or 64; `Nat` is counted by `emit_nats`, not
    here);
  - a leaf's value is a `bigint` (`Leaf = [HTerm, number, bigint, number]`; `lits_rows`, U32 only,
    converts);
  - `lits_cond` suffixes a 64-bit literal (`ull` on C, `n` on JS) and leaves 32-bit output
    byte-identical to upstream's;
  - the C match holds the word at `WORDS[adt.k]`; the JS match reads `u64_to_word(f64_bits(s))` /
    `u64_to_word(s)`.

  Checked on the three lanes with deep bit patterns over U64, I64 and F64 (`case U64{WCon{False{},
  WCon{True{}, t}}}` and the like): identical answers, identical to the omen tip where the omen tip
  compiled. **At `81f12fc` an F64 match on the C lane crashed** (`lay.arms.find` on a null); after
  the merge it compiles.
- `tsc --noEmit` over `bend2/pack`: `comp.ts` clean. The two errors it reports are in upstream's own
  `bend.ts` (lines 2171, 3882) and are there at `6a77e12` too.

## Deviations, named

1. `comp.ts` cap **lowered** 84,100 → 82,900: upstream shed 1,303 over the merge base, and the
   merged file measures 82,894. The 64-bit matcher reconciliation adds no public surface.
2. `base.bend` cap 48,928 → 49,777: exactly upstream's own +849 over the merge base (`U32.log2`,
   `Array.fork` / `Array.join`, the `Array.map` get/set loop). Measured 49,777. No margin.
3. `bend.ts` cap 42,300 → 44,000, upstream's (measured 43,780).
4. `tests/caps.sh` moved with `gates/repo.ts`, as at 2.0.17.
5. The tests `.bend` row takes upstream's `([a-z0-9-]+\/)?`; the `.c|.js` row stays at our 8,500
   (upstream 8,000); every omen row kept. Repo gate 56/56.
6. **`ttok` here is a shim.** The pip `ttok` cannot fetch `cl100k_base` through this container's
   proxy; the shim counts with `js-tiktoken`'s `cl100k_base`. It reproduces the 2.0.17 ledger's
   81,572 for `c932ae00`'s `comp.ts` exactly.

## Found, not fixed (true at `81f12fc` too; not this merge's business)

1. **`tests/run/computed_match.bend` does not build on C**: "two names mangle to CID_LT". Our
   `RUNTIME_ADTS` adds `Cmp` (and `Char`, `List`, `Inst`, `Match`) so the string natives can build
   `LT` / `EQ` / `GT`; the test's own `Ord{Lt, Eq, Gt}` upper-cases onto the same macro. Upstream
   passes it. A `cid_mac` that keeps case, or a namespace for runtime constructors, would close it.
2. **`tests/io/marshal_char_scalar.bend`** expects `char_new` to refuse 55296 on the interpreter and
   JS lanes; our `char_new` answers a `RawChar` (the strings design, pinned by
   `tests/strings/raw_strings`). A deliberate divergence of ours, now visible as an upstream test.
3. **An F64 `d` literal does not compile** on JS or C (`1.5d` alone as `main`: `lay.arms[j]` on a
   null). The interpreter prints it. The f64 suite builds its values through `F64.from_*`, so nothing
   pins it.
4. **Upstream's `gates/test.ts` cannot read our tree**: `tests/` holds files (`run.sh`, `caps.sh`,
   `t1_basic.bend`, `t_mc.bend`), and it `readdir`s every entry as a directory. Our suites also import
   `../../demos` and `../../power`, which its shard packer does not copy.
5. Upstream fixed the `parse_term_ns` bug from 2.0.17 deviation 10 (`4dfbaa1`, "An operator is the
   name whose only dot leads it"). The three `Nat.add` / `Nat.mul` sites in `demos/python` can go
   back to `( .. : Nat)`; not done here.

## Cap ledger (ttok)

| file | merge base `6018e28` | ours `81f12fc` | upstream `6a77e12` | after the merge | cap |
|---|---:|---:|---:|---:|---:|
| `bend2/base.bend` | 24,557 | 48,928 | 25,406 | 49,777 | 49,777 |
| `bend2/bend.ts` | 41,915 | 42,264 | 43,431 | 43,780 | 44,000 |
| `bend2/comp.ts` | 63,466 | 84,045 | 62,163 | 82,894 | 82,900 |

Upstream's delta over the merge base: base +849, bend.ts +1,516, comp.ts −1,303. The merge carries
all three onto ours: base and bend.ts rise by their upstream delta, comp.ts falls by upstream's
delta, and the resolution seams (the 64-bit matcher, the kept rows and comments) add back +152. Ratio of the merge over `81f12fc`:
104 files, +8,324 / −19,270 (del/ins 2.31), almost all of it upstream's effect consolidation.

## Battery

Run on this container (4 cores, no GPU, no ALSA headers, no pinned CPython oracle at
`/home/omen/.hermes/...`, no `~/Documents/Project`).

| | repo | f64 | strings | codex | regex | parser | lint | translator | caps |
|---|---|---|---|---|---|---|---|---|---|
| `d6792ca` sync | 56/56 | 18/1 | 100/2 | 161/0 | 48/1 | 108/7 | 76/1 | 40/17 | ok |

Every failure is the machine, not the merge:

- **f64 1, strings 1, regex 1**: the GPU lanes (`mc_pi`, `gpu`, regex `gpu`): "this binary found no
  GPU device".
- **strings `deep [interpret]`**, status 124: past `STRING_TIMEOUT`'s 300 s. It prints the right
  answer. The omen tip is past it here too: `81f12fc` 297 s alone, 318 s under load; the merge 314 s
  and 336 s. Swapping only `bend.ts` back to ours measures 314.5 s vs 325.8 s: upstream's evaluator
  costs this term about 3.5%.
- **parser 7, lint 1, translator 17**: the pinned-oracle checks, `FileNotFoundError` on
  `/home/omen/.hermes/hermes-agent/venv/bin/python3`. With the pin pointed at this container's
  CPython 3.11.15 in a scratch copy: parser 7/7, lint 1/1, translator 1/17. The other 16 judges read
  their sources from `~/Documents/Project/*`, which does not exist here.

Upstream's `gates/test.ts`, run locally: a scratch copy that runs the shard script with `bash`
instead of `ssh`, two shards, three builds at a time, a 20 s alarm instead of 5, only the directories
under `tests/`, and `demos/` and `power/` shipped with each shard. **1578 / 1594.** The 16:

- `codex_t_{conversions,math,precision,rounding,special,text}` (6): no `#|` block; the codex suite
  keeps its own expectations (161/0).
- `io_audio_only_{open,write,close}`, `io_audio_open` (4): no `alsa/asoundlib.h`.
- `f64_mc_pi [js]`: past the 20 s alarm (the f64 suite's JS lane passes it).
- `io_fork_join [js]`: sleep-ordered, 10 ms apart; under load `6a77e12` itself orders it 0/10 and
  the merge 5/10.
- `parser_io_read`: its 3 MiB fixture is written by `tests/parser/run.sh`.
- `strings_raw_strings [js]`: the strings suite expects the JS rejection explicitly, and passes it.
- `run_computed_match [js, c]`, `io_marshal_char_scalar [interp, js]`: found-not-fixed 1 and 2
  above; the same at `81f12fc`.

## Not done

- Nothing pushed; `bend-labs` not committed.
- Found-not-fixed 1-5 above.
- Not run: `tests/{decode,kernels,power}/run.sh` (not in the 2.0.17 battery).
