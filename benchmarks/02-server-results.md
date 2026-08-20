# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=14` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 120 | 2.15 | 3700 | 5700 | 6200 | 8.1 | 0.0% |
| 50 | 141 | 2.39 | 19000 | 21000 | 22000 | 39.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.11x** (22% of linear) |
| P95 latency | **3.68x** |
| Effective concurrency at 50 users | 39.4 vs `--parallel 4` slots (occupancy/slot ratio 9.85) |

**Saturated.** Throughput delivered only 1.11x for 5x the offered load, and effective concurrency (39.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.11x while P95 moved 3.68x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server saturates **at or below 10 users** — not somewhere between 10 and 50. The
number that convinced me is `make metrics` sampled during the 50-user run: peak
`n_busy_slots_per_decode = 3.97` of 4 `--parallel` slots, held essentially flat (3.94 →
3.97) for the full 60s sample, with `deferred` sitting at ~42-46 requests queued behind
those 4 slots the entire time. All four decode slots were continuously occupied; there
was never spare capacity for the extra 40 simulated users to use. That's confirmed by
throughput: RPS only moved from 2.15 to 2.39 (1.11x) for 5x the offered load, while P95
moved 3.68x (5700ms -> 21000ms). Since throughput barely grew, the extra latency at 50
users is almost entirely **queue time**, not extra compute time — each request still
does the same amount of prefill+decode work once it gets a slot, it just waits longer
in the deferred queue to get one. Effective concurrency (39.4, Little's Law) confirms
this from a different angle: with only 4 slots actually decoding, the other ~35
"in-flight" requests by definition can only be sitting in the queue, not running.

**What I'd change first: raise `--parallel`** (more decode slots), not thread count or
quantization. The bottleneck signature here is a queue backed up behind a fixed slot
count while the slots themselves are fully busy but not overloaded per-request (P50 at
10 users was already 3700ms for these long-RAG/short prompts, in line with the raw
per-request decode cost from `01-quickstart-results.md`). Raising `-t` wouldn't help —
`01-tuning-tg128.md` already showed threads are a second-order knob under Metal
offload. Dropping to 2-bit would shave a small amount of per-token compute (1.08x from
`01-quickstart-results.md`) but wouldn't add queue capacity, so it doesn't touch the
real bottleneck. More `--parallel` slots directly reduces `deferred`, at the cost of
splitting the same GPU compute (and KV-cache memory) across more concurrent decodes —
which is a real tradeoff worth measuring (see `bonus/sweeps/gpu-sweep.py` /
`--parallel` sweep), but it attacks the actual saturation point instead of a knob that
isn't currently the constraint.
