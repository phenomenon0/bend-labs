# monoids-next — the shortlist from mining all the deep dives (2026-09-23)

Three agents read the deep dives in full: systems (crypto, hashtables,
compression, caches, allocators, concurrency), graphics/physics (shaders,
textures, gamephysics, gamecharacter, navigation, audio, gpu) and ML/numerics
(rl, floatingpoint, rng, efficiency papers, gguf, parsers, unicode). Each pick
satisfies the power-lane rules. The leaf is pure arithmetic on data that is a
function of its index. The join is a small associative Data summary. There is
an independent oracle. The core claims were checked in CPython by the agents;
none is built yet.

## Ranked

| # | program | deep dive | the structure | summary | oracle | claim checked |
|---|---|---|---|---|---|---|
| 1 | **Linear probing without the table** | hashtables | an invertible fmix run backwards gives each slot's arrival count; the carry `c=max(0,c+a-1)` is a (max,+) affine map, composed; pass 2 descends for clusters | 2 U32 (pass 1), ~9 U32 (pass 2) | CPython really inserts; Knuth's formulas beside it | m=2^16, α=.85: 174706 displacement both ways |
| 2 | **ASF poker heist** | rng | fork over all 86.4M GetTickCount seeds; the Delphi LCG run backwards and 52 swaps traced in reverse for 5 positions, no deck | 6 U32 (count by prefix, min seed) | CPython forward shuffle | 204 seeds matched; window counts [5340,153,7,2,1] |
| 3 | **Jitter Tongues** | gamephysics | per pixel (ωa, ωb): alternating symplectic-Euler steps, each stable on its own, whose product is unstable when \|2−xy\| < \|x−y\| | 3 U32 + PGM | closed-form trace test + CPython f32 replay | x=1.6, y=1.0 reaches 1e192 in 2000 steps |
| 4 | **Lost-update census** | concurrency | every interleaving of 2×n `counter++` (C(32,16)=601M at n=8), unranked + Gosper | 4–15 U32 histogram | memoized Python DP | the minimum is 2, reached by exactly 8 schedules for every n≥3; refutes the article's "or all of them" |
| 5 | **Online softmax, exact** | attention-tax | FlashAttention's (m,l,o) merge is float addition with exponent alignment; base-2 integer logits make it an exact accumulator | 7 U32 | Fraction | F32 tree gives 5 different bits at 5 depths; the "standard" order is off by ~140 ULP |
| 6 | **Pratt as a monoid** | parsers | +,−,× evaluation as sparse 3×3 matrices mod 2^32: Prod{P} / Sum{L,M,T} | 4 U32 | eval() mod 2^32 | depths 0/3/7/11 agree with eval |
| 7 | **AES-GCM from xor/shift/and** | crypto | CTR is indexed; GHASH is (acc, H^len) over GF(2^128), carry-less so no U64; S-box computed as x^254 | 8 U32 | McGrew–Viega test vectors | not run |
| 8 | **White-furnace audit** | textures | GGX importance sampling per (μ, roughness) cell, fixed-point sum | 2 U32/cell | Heitz 2014 identities + f32 replay | the article's direct-light k keeps 0.199 of grazing light at roughness .05 (0.975 correct) |

Runners-up: the Quake-constant exhaustive check (it corrects the floatingpoint deep
dive's "0.0932%"), the Chowning/Bessel FM chart, the Shadertoy sin-hash
portability census, and minimal-perfect-hash first-seed search.

## Rejected across all three (why)

- Merkle–Damgård hashes, KDFs and memory-hard functions: sequential by design.
- rANS, LZ77 and BWT: the leaves allocate streams.
- Cache simulators: the state is not small.
- Mamba or linear-attention scans: the affine monoid again, and not exact in F32.
- UCB and Thompson sampling: they need ln or Beta draws, which differ across lanes.
- GGUF block quantization: a map with a trivial join; the dot product is Kulisch again.
- The particle filter and EKF: they need a global renormalization.
- CCD, bank conflicts, tensor-core formats: closed-form trivia.

## Free oracles (deep-dive errata found along the way)

- **xorshift32 trace (rng):** the output from 0xDEADBEEF is 0x477D20B7, 0x8E1D9142, 0xBA8C2458.
- **glibc LCG (rng):** X(2) = 377401575.
- **Dice (rng):** exactly 1000 of each face in 6000 rolls has odds of about 1 in 1.28e9.
- **Summing 10M × 1.0f (floatingpoint):** the naive float sum is exactly 1e7.
- **Summing 1e9 × 0.1f (floatingpoint):** the naive sum sticks at 2,097,152.
- **Q4_0 pseudocode (gguf):** it is not what ggml does. ggml uses d = max/−8, then min(15, x·id + 8.5).
- **Canonical shader hash (shaders):** `fract(sin(dot)·43758.5453)` disagrees in 76.5% of 8-bit cells across plausible sin implementations.
- **UE4 direct-light k in the textures BRDF:** it loses most grazing energy for near-mirrors.
