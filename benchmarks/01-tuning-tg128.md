# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **14 physical · 14 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 89.3 | 94% |
| 7 | 94.9 | 100% |
| 14 | 88.6 | 93% |
| 28 | 74.7 | 79% |

**Best**: `-t 7` at 94.9 tok/s
**Slowest tested**: `-t 28` at 74.7 tok/s (1.27x spread)
**Against the physical-core default** (`-t 14`, 88.6 tok/s): 1.07x

Use this in your run:

```bash
LAB_N_THREADS=7 make bench
```

## Your explanation

This run has `ngl=99` — every layer is offloaded to the M4 Pro's Metal GPU, so the
decode matmuls (the FLOPs that CPU thread count would normally parallelize) are not
running on the CPU at all. That's why the curve does **not** look like a classic
CPU-bound thread sweep: `-t 1` (89.3) and `-t 14` (88.6) are within ~1% of each other,
a gap small enough to be noise, not compute scaling. If this were CPU-only decode, 14
threads should beat 1 thread by close to 14x on the compute-bound portion; here it
doesn't beat it at all, which is itself the evidence that GPU offload — not the thread
count — is doing the decode work.

The knee is not at the physical core count (14) the way the deck's CPU-only intuition
would predict. It's a shallow bump at `-t 7` (94.9, +6% over `-t 1`), then a real
21% drop at `-t 28`. The CPU threads here aren't running the matmuls — they're running
the per-token host-side loop between GPU dispatches (sampling, KV-cache bookkeeping,
scheduling the next Metal command buffer). A handful of threads (~7) is enough to keep
that loop pipelined without idling; the small gain over `-t 1` is that pipelining, not
FLOPs parallelism. `-t 28` asks for twice as many threads as the chip has physical
cores, so the scheduler has to time-slice across all of them for a workload where the
useful work per token step is tiny and serialized (each thread is waiting on the GPU
call before it can do anything). Oversubscription there adds context-switch and
scheduling overhead directly onto the critical path with zero compute benefit, which
is exactly the 21% drop.

**Contradicts the deck's expected shape** (peak flat until physical core count, then
drop) because this machine runs GPU-offloaded, not CPU-only, decode. The mechanism that
sets the deck's expectation — memory-bandwidth-bound CPU matmuls scaling with thread
count up to core count — doesn't apply once ngl=99 moves that bandwidth pressure onto
the GPU's own unified-memory path. The takeaway for this hardware: thread count is a
second-order knob under Metal offload (a 1.07x spread, `-t 7` vs `-t 14` default);
what would actually move the needle by a large margin is something that changes GPU
work, like quantization (§2) or batch/parallel slot count, not `-t`.
