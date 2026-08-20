# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.97 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 14559 |

Highest sampled value was **3.97 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width was **3.97 of 4 slots** — essentially every decode step during the
60s sample had all 4 `--parallel` slots occupied. That does **not** match the 39.4
"effective concurrency" figure in `02-server-results.md`, but the two aren't actually
in conflict — they measure different things. `n_busy_slots_per_decode` is the server's
own gauge of how many slots are *actively decoding* at once, so it is hard-capped at
`--parallel` (4) by construction. Effective concurrency (`RPS x average latency`,
Little's Law) counts every request currently *in the system*, including the ~42-46
sitting in `requests_deferred` waiting for a free slot — it has no such cap, which is
exactly why it can legitimately read 39.4 while only 4 requests are ever decoding at
once.

**I trust `n_busy_slots_per_decode` for "is the server's compute capacity saturated"**
— it's a direct, ground-truth server counter, not a derived estimate, and 3.97/4
answers that question unambiguously (yes). **I trust the 39.4 effective-concurrency
number for "how much demand is the system carrying overall"** — it's the one that
explains why P95 blew up 3.68x even though only 4 requests decode at a time: the other
~35 are queued, and queue depth is exactly what Little's Law is measuring. Read
together rather than against each other: 3.97/4 says the server is at its serving
limit, and 39.4 says demand is running ~10x past that limit, with the surplus sitting
in `requests_deferred` accruing queue time.
