# Bonus C1 - Speculative decoding with Gemma 4's MTP head

Host `Darwin-arm64` · CPU `Apple M4 Pro` · backend `apple_metal` (`ngl=99`)
llama.cpp `b10488` · target `gemma-4-E2B-it-UD-Q4_K_XL.gguf` ·
draft `mtp-gemma-4-E2B-it.gguf` (`--spec-type draft-mtp`, single request, no load)

Flags used on this build (`llama-server --help | grep -iE "draft|mtp|spec"`):
`-md/--model-draft` for the draft model path, `--spec-type draft-mtp` to select the
MTP speculative mode. (The CHALLENGES.md doc names `-md`/`--draft-max` for build
`b10488`; `--draft-max` has since been removed in favor of `--spec-draft-n-max`, and
`--spec-type` must be set explicitly to pick `draft-mtp` — flag names moved between
llama.cpp revisions even within the same pinned build's docs, confirming the doc's own
warning to check `--help` first.)

| Temperature | Spec decode | predicted tok/s | tokens generated |
|--:|:--|--:|--:|
| 0.0 | OFF (baseline) | 87.9 | 111 |
| 0.0 | ON (MTP draft) | 61.5 | 111 |
| 0.8 | OFF (baseline) | 82.1 | 125 |
| 0.8 | ON (MTP draft) | 58.4 | 121 |

**Result: speculative decoding was slower here, not faster** — 0.70x at temp 0, 0.71x
at temp 0.8. This is the opposite direction from the deck's EAGLE-3 headline number
(3-6.5x).

**Draft acceptance**, read from the server's own per-request log line
(`slot print_timing: ... draft acceptance = ...`) over 3 requests during this run:

```
draft acceptance = 0.345  (57 accepted / 165 generated), mean accepted len = 2.04
draft acceptance = 0.302  (58 accepted / 192 generated), mean accepted len = 1.91
draft acceptance = 0.341  (47 accepted / 138 generated), mean accepted len = 2.02
```

## Why it came out this way

Acceptance rate is only ~30-35%, meaning roughly 2 out of 3 draft tokens are rejected
and thrown away. Every draft round still costs a full forward pass of the *target*
model to verify (that's unavoidable — verification is what guarantees the output
matches greedy/sampled decoding from the target alone), plus an extra forward pass of
the draft (MTP head) model to propose tokens in the first place. On this hardware the
target model is already decoding fast on its own (82-88 tok/s, single request, fully
Metal-offloaded, per `01-quickstart-results.md`/`bonus-gpu-offload-sweep.md`) — there
isn't much idle GPU time for a draft pass to exploit "for free" the way speculative
decoding does when the target is memory-bandwidth-bound and underutilized. With a
~2-token mean accepted length, the best case per draft round is "skip 2 verify steps",
but that gain is being paid for by the draft model's own forward pass *and* the
wasted compute on the ~65% of drafted tokens that get rejected and recomputed anyway.
When acceptance is this low and the target is already fast standalone, the
overhead outweighs the tokens saved — net negative, exactly what a rejected-token
accounting predicts, not a bug.

This also matches the CHALLENGES.md framing: EAGLE-3's 3-6.5x is measured against
larger, slower target models where decode is the dominant cost and idle accelerator
time between token-by-token verifications is large. A 2B target that's already fast on
a single request (this lab's whole point per `hardware.json`'s `ngl=99` Metal path)
is close to the regime the challenge doc calls out — "production engines often disable
speculative decoding above a batch-size threshold" — except here the threshold is
crossed at batch size 1 already, because the base decode is fast enough that
verification overhead dominates from the start. I did not additionally run this under
`make load-50` (concurrency); given the single-request result is already net-negative,
I'd expect concurrent load to make it worse, not better — the draft model's own
forward passes would compete with target-model decode for the same 4 `--parallel`
slots' GPU time, adding contention on top of an already-losing tradeoff. That is an
extrapolation from the mechanism above, not a separate measurement.
