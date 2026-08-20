# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=14` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 2044 | 74 / 209 | 11.6 / 11.8 | 795 / 944 / 944 | 86.4 |
| UD-Q2_K_XL | 2.24 | 2024 | 73 / 288 | 10.8 / 11.5 | 752 / 1011 / 1011 | 93.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.08x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

`UD-Q2_K_XL` is only **1.08x** faster on decode (93.0 vs 86.4 tok/s) and 0.73 GB smaller
on disk than `UD-Q4_K_XL`. On this machine (M4 Pro, 48 GB unified memory, ngl=99 so both
quants run fully on the GPU), that gap is small because decode is bounded by memory
bandwidth on unified-memory Metal, and 2.24 GB vs 2.97 GB of weight traffic per token
step is not a big enough delta to matter much next to fixed per-token overhead. TTFT P95
is actually *worse* on the 2-bit model (288ms vs 209ms) — noise from small samples (only
10 requests each), not a real prefill regression, since P50 TTFT is nearly identical
(73ms vs 74ms).

I asked both quants the same question ("why is TPOT bounded by memory bandwidth rather
than FLOPs during decoding?") at temperature 0. Both gave a correct, coherent two-sentence
answer that named memory bandwidth as the bottleneck and referenced sequential weight
access. I could not tell the two apart on this factual/technical prompt.

**Verdict: not worth it here.** With 48 GB of RAM and disk to spare, the ~0.7 GB saved by
dropping to 2-bit buys a ~7% throughput gain and (per Unsloth's own quantization docs)
real risk of quality loss on harder tasks — reasoning, math, long-context — that a single
short factual prompt won't surface. On a RAM-constrained machine (4-8 GB) where Q2 is the
difference between fitting in memory or swapping, the calculus flips completely; the 2-bit
quant would win outright regardless of the small quality tax. On this hardware, 4-bit is
the better default.
