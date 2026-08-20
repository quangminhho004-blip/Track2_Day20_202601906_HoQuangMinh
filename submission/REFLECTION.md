# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Hồ Quang Minh (MSSV 2A202601906)
**Cohort:** 4
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS (Darwin 25.5.0), arm64
- **CPU:** Apple M4 Pro
- **Cores:** 14 physical / 14 logical
- **CPU extensions:** NEON
- **RAM:** 48.0 GB
- **Accelerator:** Apple Metal (built into the release binary), `ngl=99`
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary, 2.97 GB) + UD-Q2_K_XL (compare, 2.24 GB)

**Chạy ở đâu:** laptop của tôi (local, không dùng cloud fallback — RAM 48GB thừa yêu
cầu 8GB).

**Setup story** (≤ 80 chữ): Máy chỉ có Python 3.9.6 hệ thống (không có brew/cmake), nhưng
`make setup` vẫn chạy được vì Makefile không đòi `python_requires>=3.10` từ pyproject
(chỉ `pip install -r requirements.txt`). Không có bước nào fail; setup + 5.2GB model
tải xong sạch trong một lần chạy.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 2044 | 74 / 209 | 11.6 / 11.8 | 795 / 944 / 944 | 86.4 |
| UD-Q2_K_XL | 2.24 | 2024 | 73 / 288 | 10.8 / 11.5 | 752 / 1011 / 1011 | 93.0 |

**Quan sát** (≤ 60 chữ): Q2 chỉ nhanh hơn Q4 1.08x (93.0 vs 86.4 tok/s), nhẹ hơn 0.73GB.
Đã hỏi cùng câu ở cả hai (`--compare`, temp=0): cả hai trả lời đúng, không phân biệt
được chất lượng trên câu hỏi factual ngắn. Với 48GB RAM dư dả, mức tăng tốc nhỏ không
đáng đánh đổi rủi ro chất lượng trên task khó hơn — Q4 là default hợp lý ở đây.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.15 | 3700 | 5700 | 6200 | 8.1 | 0.0% |
| 50 | 2.39 | 19000 | 21000 | 22000 | 39.4 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.11×
- **P95 tăng:** 3.68×
- **Effective concurrency ở 50 users:** 39.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.97 / 4 slots

**Saturation reading** (≤ 80 chữ): Bão hoà ở ngay 10 users, không phải giữa 10 và 50.
Bằng chứng thuyết phục nhất: busy_slots giữ ở 3.94-3.97/4 suốt 60s load-50, với
~42-46 request bị deferred liên tục — 4 slot luôn bận, không có chỗ trống cho tải
thêm. Throughput chỉ tăng 1.11x trong khi P95 tăng 3.68x → phần latency thêm gần như
toàn bộ là queue time (chờ slot), không phải compute time (mỗi request vẫn tốn cùng
lượng compute khi tới lượt). Muốn tăng goodput@SLO, tôi sẽ tăng `--parallel` trước —
đúng knob vì nghẽn hiện tại là số slot, không phải tốc độ per-token (threads đã là
knob bậc hai dưới Metal offload theo §5, và đổi quantization chỉ giảm nhẹ compute chứ
không thêm capacity xếp hàng).

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | — | không có trong pipeline này |
| N17 Data pipeline | TOY_DOCS (3-doc toy corpus) | stub |
| N18 Lakehouse | — | không có trong pipeline này |
| N19 Vector + features | keyword-overlap `retrieve()`, `embed()` skip | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms (skip — không chạy `serve-embed`, không phải embedding "miễn phí")
- retrieve: 0.0 ms (keyword match in-process trên 3 doc, dưới độ phân giải của timer)
- llm: 449.6 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): llm chiếm 100% vì đây là stage duy nhất làm việc thật — N19
là keyword match rẻ, N17 là embedding bị skip. Không hẳn khớp kỳ vọng "RAG thật" (nơi
retrieve + embed cũng tốn), nhưng đúng với pipeline đã stub. Muốn giảm 2x: tấn công
prefill của llm (giới hạn độ dài context retrieve, prompt caching cho prefix chung),
không phải decode — decode đã ~11-12ms/token, ít dư địa dưới Metal offload.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** `-t` (thread count) từ default 14 (physical core count) xuống `-t 7`, đo
bằng `make tune` (`llama-bench`, metric tg128, `ngl=99`)

```
before:  88.6 tok/s   (-t 14, physical-core default)
after:   94.9 tok/s   (-t 7, best trong sweep)
speedup: 1.07×
```

**Tại sao nó work** (1–2 đoạn):

Máy này chạy `ngl=99` — mọi layer offload lên Metal GPU của M4 Pro, nên các phép nhân
ma trận của decode (thứ mà thread count CPU thường parallelize) không chạy trên CPU
nữa. Đó là lý do curve không giống thread sweep CPU-only cổ điển: `-t 1` (89.3) và
`-t 14` (88.6) chỉ chênh ~1%, gần như noise, không phải compute scaling — nếu decode
compute-bound trên CPU, 14 thread phải nhanh hơn 1 thread gần 14 lần. Các CPU thread ở
đây chạy vòng lặp host-side giữa các lần dispatch GPU (sampling, bookkeeping KV-cache,
xếp lệnh Metal command buffer tiếp theo); một số ít thread (~7) đủ để pipeline vòng
lặp đó mà không bị idle, đó là nguồn của mức tăng 6% nhỏ. `-t 28` giảm 21% vì yêu cầu
gấp đôi số core vật lý, buộc scheduler time-slice trên một workload mà mỗi thread
phần lớn thời gian chỉ chờ GPU trả kết quả — oversubscription cộng thẳng overhead
chuyển ngữ cảnh vào critical path mà không có lợi ích compute nào bù lại.

**Kết quả này khác kỳ vọng từ deck**: deck mô tả peak ở physical core count rồi giảm
khi vượt qua — đúng cho decode chạy trên CPU. Ở đây peak lệch xuống `-t 7` (nửa số core
vật lý) và toàn bộ dải chênh lệch chỉ 1.27x (so với spread lớn hơn nhiều nếu CPU-bound),
vì cơ chế mà deck giả định — matmul memory-bandwidth-bound scale theo thread count đến
core count — không áp dụng khi Metal offload đã chuyển áp lực bandwidth đó sang đường
unified-memory riêng của GPU. Bài học cho phần cứng này: thread count là knob bậc hai
dưới GPU offload (chỉ 1.07x); thứ thật sự tạo speedup lớn sẽ là thứ đổi khối lượng công
việc GPU — quantization, hay số slot `--parallel` (xem §3, §6) — không phải `-t`.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B1 (`build-llama` + `compare-builds`) · B2 (`sweep-gpu`) · B4 (C1 — MTP
speculative decoding) · B5 (C8 — semantic cache, real chat+embedding servers)

**Numbers** (B2, GPU offload sweep — the clean, monotonic result; see
`benchmarks/bonus-gpu-offload-sweep.md`):

```
before:  39.3 tok/s   (-ngl 0, CPU only)
after:   88.9 tok/s   (-ngl 99, full Metal offload)
speedup: 2.26×
```

**Điều này nói lên gì mà deck chưa nói:**

Bốn thí nghiệm bonus kể cùng một câu chuyện từ bốn góc khác nhau: **trên phần cứng
này, thứ quyết định tốc độ là có offload lên GPU hay không, không phải cách compile.**
B2 cho con số sạch nhất: 2.26x từ CPU-only sang full Metal offload, tăng đơn điệu
không có điểm gãy vì unified memory không có ngưỡng VRAM để tràn (xem file bonus). B1
lại cho kết quả ngược kỳ vọng — build "native" cho đúng CPU này (`-DGGML_NATIVE=ON`)
**chậm hơn** bản prebuilt 0.82x, vì NEON đã là baseline bắt buộc trên mọi binary
arm64 macOS (không giống AVX2/AVX-512 trên x86, không có ISA rộng hơn để "native"
mở khoá), nên cờ compile không có nhiều dư địa để ăn điểm trên chip này.

C1 (MTP speculative decoding) cũng cho kết quả âm — chậm hơn baseline 0.70-0.71x —
vì tỉ lệ chấp nhận draft chỉ ~30-35% (đo trực tiếp từ log server), nên chi phí verify
+ forward pass của draft model vượt quá lượng compute tiết kiệm được, nhất là khi
target model vốn đã decode nhanh (single request, Metal). Đây đúng là trường hợp
CHALLENGES.md cảnh báo — "production engine tắt spec decode ở batch size cao" — chỉ
khác là ở đây ngưỡng đó bị vượt ngay từ batch size 1.

C8 (semantic cache) cho một insight khác hẳn về *chất lượng*, không phải tốc độ: cache
dùng embedding từ chat model pooled (không phải embedding model chuyên dụng) vừa bỏ
sót true paraphrase (#3, #6 chỉ đạt sim 0.72-0.80, dưới ngưỡng 0.85) vừa false-hit một
topic hoàn toàn mới (#7, sim 0.85). Không có ngưỡng nào sửa được cả hai lỗi cùng lúc —
đó là bằng chứng cụ thể cho lý do vì sao cần embedding model chuyên dụng, không phải
tái dùng chat model, cho semantic cache thật.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Speculative decoding (C1) làm decode **chậm hơn** (0.70x), không nhanh hơn — trái
ngược hoàn toàn với con số 3-6.5x của EAGLE-3 trong deck. Bất ngờ thứ hai: build từ
source "tối ưu cho đúng CPU của tôi" lại thua bản prebuilt (0.82x), vì trên Apple
Silicon NEON đã là sàn bắt buộc chứ không phải thứ `-DGGML_NATIVE` có thể mở khoá
thêm — hai lần liên tiếp "làm nhiều hơn" (thêm draft model, compile riêng cho CPU)
hoá ra chậm hơn "làm ít hơn" (baseline, prebuilt binary).

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
