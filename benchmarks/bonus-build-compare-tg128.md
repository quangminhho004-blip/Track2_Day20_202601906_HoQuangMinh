# Bonus B1 - Prebuilt vs source build

Host `Darwin-arm64` · CPU `Apple M4 Pro`
Vector extensions detected: NEON
llama.cpp `b10488` both sides · `threads=14` ·
**both pinned to `ngl=0`** so this isolates the compiler ·
metric `tg128`, 3 repetitions

| Binary | Built for | tg128 (tok/s) | Relative |
|:--|--:|--:|--:|
| prebuilt release | runtime CPU dispatch | 52.8 | 1.00x |
| your source build | this CPU (`-DGGML_NATIVE=ON`) | 43.1 | 0.82x |

On this machine, the prebuilt binary is **1.22x faster**.

before: 52.8 tok/s (prebuilt release)
after:  43.1 tok/s (source build, -DGGML_NATIVE=ON)
speedup: 0.82x

Same source revision, same model, same backend, same `-ngl` -- the only difference
is what the compiler was allowed to assume about the CPU.


### Separately: what GPU offload is worth on the same binary

`tg128` on the source build at `-ngl 99` instead of `-ngl 0`:

| Source build | tg128 (tok/s) | vs its own CPU run |
|:--|--:|--:|
| `-ngl 0` (CPU) | 43.1 | 1.00x |
| `-ngl 99` (offloaded to MTL0: Apple M4 Pro (38338 MiB, 38338 MiB free)) | 88.7 | 2.06x |

This number is **not** part of the B1 comparison above -- it is a different knob.
Reporting it separately is the point: a compiler flag and an accelerator are not
interchangeable explanations for a speedup.


## Your explanation

The prebuilt binary won, 52.8 vs 43.1 tok/s (0.82x for my "native" build) — the opposite
of what B1 usually finds. This is possible, and I think it comes down to `NEON` being
the *only* vector extension listed for this CPU. On x86, `-DGGML_NATIVE=ON` is a big
lever because the compiler can go from a conservative SSE baseline up to AVX2/AVX-512
if the host supports it — a real, large instruction-set upgrade. On Apple Silicon, NEON
is not optional: every arm64 macOS binary, prebuilt or not, is compiled against NEON
already, because it's the universal baseline for the architecture. So `-DGGML_NATIVE=ON`
here had no wider vector ISA to unlock — the only thing left for it to change is
`-mcpu` scheduling/tuning hints for the M4 Pro specifically, and that's a much smaller,
riskier lever: AppleClang's scheduling model for a very recent chip (M4 Pro) may be
less mature than whatever generic/older Apple Silicon tuning target the official
release CI builds against and has validated across many machines. A local `-O3`
Release build with no extra tuning flags (no explicit LTO/IPO, which the Makefile's
`build-llama` target doesn't set) is also plausibly just a slightly worse-optimized
binary than whatever build flags the upstream release pipeline uses.

This is also consistent with `tg128` (decode) being memory-bandwidth-bound, not
instruction-bound (established in `01-tuning-tg128.md` §5): when the bottleneck is
how fast weights move from memory, not how many FLOPs the ALU issues per cycle,
compiler-level codegen differences have limited room to help *or* hurt — a ~20% swing
either direction from CPU-target tuning is believable noise around a bandwidth ceiling,
not a sign the source build is badly broken. The GPU-offload number confirms where
the real headroom is: `-ngl 99` on the same source build gets 88.7 tok/s, a 2.06x jump
that dwarfs anything a compiler flag produced. **Takeaway: on Apple Silicon specifically,
"compile native for your CPU" is not a safe assumption of a win the way it is on x86 —
NEON is already the floor, not an optional target, so B1's real lesson here is that the
official prebuilt binary is already well-tuned and a from-source build doesn't
automatically beat it.**
