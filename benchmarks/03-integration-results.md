# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 558.1 | 558.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 387.9 | 388.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 402.8 | 402.8 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **449.6** · total **449.6**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC** — not present in this pipeline run; not exercised by `make pipeline`.
- **N17 Data pipeline** — not present; the 3 queries use a fixed 3-doc toy corpus
  (`TOY_DOCS` in `labs/03-integrate/pipeline.py`), no ingestion pipeline runs.
- **N18 Lakehouse** — not present, same reason as N17.
- **N19 Vector + features** — **stub**. `retrieve()` is a keyword-overlap match over
  `TOY_DOCS`, not a real vector index (no ANN, no embedding-based similarity search).
  `embed()` returns `None` because no embedding server was running for this pipeline
  run (`make serve-embed` was not started), so `embed(ms)` reads 0.0 because it's
  skipped entirely, not because embedding is free.
- **N20 Serving** — **real**. `llm(ms)` is genuine `llama-server` latency over HTTP
  (`/v1/chat/completions`), same server used for `make bench` / `make load-*`.

Stubbing N16-N19 costs no points here — this run is honest about it: the retrieval and
data layers are toy stand-ins, only the serving layer is a real inference stack.

**Is `llm` at 100% of total expected?** Yes, and for a mechanical reason, not just
because the other stages are slow-adjacent: `retrieve()` is in-process Python keyword
matching over 3 short documents, which is sub-millisecond and rounds to 0.0ms at this
timer's resolution, and `embed()` is skipped outright (no embedding server). So `llm`
isn't "dominant" the way it would be in a real RAG stack — it's the *only* stage doing
real work in this configuration. A real N19 vector index (ANN search over a large
corpus) or a real N17 embedding call over HTTP would show up as non-zero, and long-RAG
prompts would additionally push up the `llm` stage's own TTFT via a longer prefill.

**If I had to halve this pipeline's latency**, I'd attack the `llm` stage specifically
via TTFT/prefill, not decode — `01-quickstart-results.md` shows TTFT is small for short
prompts, but the moment a real N19 retrieves several long chunks of context, prefill
(compute-bound, scales with prompt tokens) is exactly where a real RAG pipeline gets
"thổi phồng" per `labs/03-integrate/README.md`. Concretely: cap retrieved-context length,
or use prompt caching for the shared system/instruction prefix, rather than trying to
speed up decode (`llm`'s decode portion is already ~11-12ms/token per the tuning run,
and Metal offload leaves little headroom there compared to cutting prefill tokens).
