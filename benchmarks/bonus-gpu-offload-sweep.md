# Bonus - GPU offload sweep

Host `Darwin-arm64` · backend(s) `apple_metal` ·
llama.cpp `b10488` · `threads=14` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 39.3 | 1.00x | 44% |
| 8 | 53.4 | 1.36x | 60% |
| 16 | 59.0 | 1.50x | 66% |
| 24 | 72.0 | 1.83x | 81% |
| 32 | 74.5 | 1.89x | 84% |
| 99 | 88.9 | 2.26x | 100% |

Best: `-ngl 99` at 88.9 tok/s
-- 2.26x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Yes — full offload (`-ngl 99`) is best on this machine, and the curve is monotonically
increasing all the way to it (39.3 -> 53.4 -> 59.0 -> 72.0 -> 74.5 -> 88.9 tok/s), never
flattening or dropping before full offload. That shape rules out "ran out of VRAM":
the M4 Pro has 48 GB of **unified** memory shared between CPU and GPU (not a discrete
GPU with its own separate VRAM pool), and `hardware.json`/`make probe` reported 38338
MiB free on the Metal device for a 2.97 GB model — there was never a capacity ceiling
to hit, so nothing "did not fit."

What the curve actually shows is each additional offloaded layer directly buying back
memory-bandwidth-bound decode work that would otherwise run on the CPU: each `-ngl`
step moves another chunk of the model's per-token weight traffic from the CPU path
(cores + shared bandwidth, contended with everything else on the SoC) onto the GPU's
own compute/bandwidth path. Because M4 Pro uses unified memory, there's no PCIe-style
host<->device transfer tax for each layer moved — offloading is closer to "which engine
reads these weights" than "copy the weights somewhere else." That's why the gains are
smooth and additive rather than showing a sharp jump-then-plateau: there's no discrete
transfer bottleneck to saturate before the ceiling, just steadily fewer layers left
paying the (slower) CPU-path bandwidth cost. On a discrete-GPU machine with limited
VRAM, I'd expect this curve to flatten or dip once the model stopped fitting and the
accelerator started re-fetching evicted layers — the unified-memory architecture here
is exactly why that failure mode doesn't show up.
