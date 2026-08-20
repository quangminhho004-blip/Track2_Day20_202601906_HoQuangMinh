# Bonus B5/C8 - Semantic cache

Real mode: chat server on `:8080` (Gemma 4 E2B, `llama-server`) + embedding server
on `:8081` (`labs/02-serve/serve.py --embedding`, same model in pooling mode — this
lab ships one model, so the "embedder" here is mean-pooled decoder hidden states,
not a dedicated embedding model). Threshold = 0.85.

```
 #  result   sim      ms  prompt
 1  miss    0.00    1371  What is goodput at SLO?
 2  miss    0.55    1000  Explain TTFT and TPOT.
 3  miss    0.72     971  Can you define goodput@SLO?
 4  miss    0.76     982  What does time to first token mean?
 5  miss    0.64     984  How does PagedAttention work?
 6  miss    0.80    1002  Tell me what goodput@SLO is.
 7  HIT     0.85       0  What is prefix caching?
 8  HIT     0.85       0  Describe how PagedAttention works.

Hit rate: 2/8 = 25%   (threshold 0.85)
LLM calls saved: 2
```

**Ground truth:** #3 and #6 paraphrase #1; #4 paraphrases #2; #8 paraphrases #5. #7
("What is prefix caching?") is a genuinely new topic and must never hit.

## What the table actually says

Two hits, and they land on opposite sides of correctness. **#8 is a true hit** — it
correctly recognized "Describe how PagedAttention works." as a paraphrase of #5 and
returned the cached answer for zero compute. **#7 is a false hit** — a brand-new topic
("prefix caching") scored >=0.85 against something already in the cache and got served
a wrong, stale answer instead of running inference. Meanwhile the *actual* paraphrases
of #1 and #2 (#3, #4, #6) all **missed** — sim 0.72-0.80, just under the 0.85 bar —
so the cache is simultaneously letting a false positive through and rejecting true
positives. Raising the threshold to fix the false hit on #7 would push #6 (0.80)
further from ever hitting; lowering it to catch #3/#4/#6 would only invite more false
hits like #7. No single threshold fixes both failure modes here — that's the finding,
not a tuning miss.

**Why:** the embedder is a mean-pooled decoder model, not a model trained for semantic
similarity. Mean-pooling a chat model's hidden states produces embeddings that cluster
mostly by surface form (shared words, similar sentence structure, similar topic-word
co-occurrence) rather than by meaning — "What is prefix caching?" apparently shares
enough surface-level structure with an unrelated cached question to clear 0.85, while
"Tell me what goodput@SLO is." (a real paraphrase, but phrased as an imperative instead
of a question) falls short. A real embedding model (Qwen3-Embedding, BGE-M3,
EmbeddingGemma — purpose-trained with a contrastive objective to separate
paraphrases from strangers) would be expected to widen that gap in both directions:
push true paraphrases up, push unrelated prompts down, so a single threshold could
plausibly separate them. This lab shipping one model makes the "embedding server"
technically real (a running `/v1/embeddings` endpoint, real HTTP round trip, real
95%+ compute savings on an actual hit — rows 7 and 8 show 0ms vs ~1000ms for a miss),
but the *quality* of what it embeds is the wrong tool for the job — exactly the
gap the lab's own warning banner calls out before the table.

**Risk callout (from the script's teaching notes, not separately measured here):** a
shared semantic/KV cache across tenants can leak prompt content across users via
timing side channels (cache hit = fast response = "someone asked this before") —
production semantic caches must salt cache keys per-tenant. Not exercised in this run
(single-user demo), noted because it's the other failure mode C8 calls out besides
threshold miscalibration.
