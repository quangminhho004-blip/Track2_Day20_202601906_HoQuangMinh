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

**Chạy ở đâu:** em chạy local trên laptop của mình, không cần dùng cloud fallback vì
48GB RAM đã dư quá nhiều so với yêu cầu 8GB.

**Setup story** (≤ 80 chữ): Máy em chỉ có Python 3.9.6 hệ thống, không có brew hay
cmake sẵn. Em cứ nghĩ `make setup` sẽ báo lỗi vì pyproject ghi `python_requires>=3.10`,
nhưng hoá ra không sao — Makefile chỉ chạy `pip install -r requirements.txt` chứ không
`pip install -e .`, nên bản 3.9 vẫn ăn. Setup và tải 5.2GB model trôi chảy, không vướng
bước nào.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 2044 | 74 / 209 | 11.6 / 11.8 | 795 / 944 / 944 | 86.4 |
| UD-Q2_K_XL | 2.24 | 2024 | 73 / 288 | 10.8 / 11.5 | 752 / 1011 / 1011 | 93.0 |

**Quan sát** (≤ 60 chữ): Q2 chỉ nhanh hơn Q4 có 1.08x (93.0 vs 86.4 tok/s), nhẹ hơn có
0.73GB — không đáng kể lắm. Em thử hỏi cùng một câu ở cả hai bản (`--compare`, temp=0)
thì cả hai đều trả lời đúng, không phân biệt được chất lượng khác nhau chỗ nào trên
câu hỏi factual ngắn này. Máy em 48GB RAM dư dả nên mức tăng tốc nhỏ như vậy không
đáng để đánh đổi rủi ro tụt chất lượng ở những task khó hơn — em vẫn chọn Q4 làm default.

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

**Saturation reading** (≤ 80 chữ): Theo em thì server bão hoà ngay từ 10 users rồi,
không phải đợi đến giữa 10 và 50 mới bão hoà. Bằng chứng rõ nhất là busy_slots giữ
nguyên ở 3.94-3.97/4 suốt 60 giây chạy load-50, cộng thêm ~42-46 request cứ bị
deferred liên tục — nghĩa là 4 slot lúc nào cũng bận, chẳng còn chỗ trống nào cho tải
thêm cả. Throughput chỉ nhích 1.11x trong khi P95 lại tăng tới 3.68x, nên gần như toàn
bộ phần latency dôi ra đó là thời gian chờ slot chứ không phải compute — mỗi request
vẫn tốn đúng chừng đó compute khi tới lượt nó. Nếu phải tăng goodput@SLO, em sẽ tăng
`--parallel` trước tiên, vì nghẽn ở đây là số slot chứ không phải tốc độ per-token
(threads đã là knob bậc hai dưới Metal offload như em nói ở §5, còn đổi quantization
thì chỉ giảm nhẹ compute chứ không thêm capacity để xếp hàng).

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

**Reflection** (≤ 60 chữ): llm chiếm trọn 100% cũng dễ hiểu thôi, vì đây là stage duy
nhất thật sự làm việc — N19 chỉ là keyword match rẻ tiền, còn N17 (embedding) thì bị
skip luôn. Nó không giống một pipeline RAG "thật" (nơi retrieve và embed cũng ngốn
thời gian), nhưng đúng với cái pipeline đã stub sẵn ở đây. Nếu phải giảm latency 2x,
em sẽ nhắm vào phần prefill của llm trước — giới hạn độ dài context retrieve, hoặc
prompt caching cho phần prefix chung — chứ không đụng vào decode, vì decode đã
~11-12ms/token rồi, dưới Metal offload thì cũng chẳng còn nhiều dư địa để cắt nữa.

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

Máy em chạy `ngl=99`, tức là mọi layer đều được đẩy lên Metal GPU của M4 Pro, nên các
phép nhân ma trận trong decode — cái mà thread count CPU vẫn hay parallelize — không
còn chạy trên CPU nữa. Đó cũng là lý do vì sao đường cong sweep của em không giống
kiểu thread sweep CPU-only thường thấy: `-t 1` (89.3) và `-t 14` (88.6) chỉ lệch nhau
khoảng 1%, gần như là nhiễu chứ không phải do compute scale theo thread. Nếu decode mà
compute-bound trên CPU thật thì 14 thread phải nhanh hơn 1 thread gần 14 lần chứ không
phải chỉ nhích nhẹ như vậy. Ở đây các CPU thread chỉ chạy cái vòng lặp phía host giữa
các lần gọi GPU thôi — sampling, ghi sổ KV-cache, xếp lệnh cho Metal command buffer kế
tiếp — nên chỉ cần một số ít thread (khoảng 7) là đủ để pipeline vòng lặp đó chạy trơn
tru không bị idle, và mức tăng 6% em thấy ở `-t 7` chính là từ đó. Còn `-t 28` thì tụt
hẳn 21%, vì em yêu cầu gấp đôi số core vật lý, buộc scheduler phải time-slice trên một
workload mà phần lớn thời gian mỗi thread chỉ ngồi chờ GPU trả kết quả — oversubscribe
kiểu này cộng thẳng overhead chuyển ngữ cảnh vào critical path mà chẳng đổi lại được
gì về compute.

**Kết quả này khác với kỳ vọng từ deck**: deck mô tả đỉnh nằm ở đúng số core vật lý rồi
tụt dần sau đó — đúng cho trường hợp decode chạy trên CPU. Còn ở máy em, đỉnh lại lệch
xuống `-t 7` (chỉ bằng nửa số core vật lý), và cả dải chênh lệch cũng chỉ có 1.27x
thôi — nhỏ hơn nhiều so với mức spread mà một workload CPU-bound thật sự sẽ cho. Lý do
là cơ chế mà deck giả định (matmul memory-bandwidth-bound, scale theo thread count đến
hết số core) không còn áp dụng nữa một khi Metal offload đã chuyển hết áp lực bandwidth
đó sang đường unified-memory riêng của GPU. Bài học em rút ra cho phần cứng này là:
thread count chỉ là một knob bậc hai dưới GPU offload (có 1.07x thôi), còn cái thật sự
tạo ra speedup lớn sẽ là thứ nào đó đổi được khối lượng công việc GPU — như quantization,
hay số slot `--parallel` (xem thêm §3, §6) — chứ không phải `-t`.

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

Bốn thí nghiệm bonus em làm hoá ra lại kể cùng một câu chuyện, chỉ nhìn từ bốn góc
khác nhau: **trên phần cứng của em, thứ quyết định tốc độ là có offload lên GPU hay
không, chứ không phải compile kiểu gì.** B2 cho con số sạch nhất — 2.26x khi đi từ
CPU-only sang full Metal offload, tăng đơn điệu không có điểm gãy nào, vì unified
memory ở đây không có ngưỡng VRAM để tràn (chi tiết em ghi trong file bonus). Còn B1
thì lại cho kết quả ngược hẳn với những gì em nghĩ ban đầu — build "native" riêng cho
đúng con CPU này (`-DGGML_NATIVE=ON`) lại **chậm hơn** bản prebuilt tới 0.82x. Lý do
là NEON vốn đã là baseline bắt buộc trên mọi binary arm64 macOS rồi, không giống
AVX2/AVX-512 bên x86 — không có ISA rộng hơn nào để cờ "native" mở khoá thêm, nên trên
con chip này, compile riêng gần như chẳng có gì để ăn điểm cả.

C1 (MTP speculative decoding) cũng ra kết quả âm — chậm hơn baseline 0.70-0.71x — vì
tỉ lệ chấp nhận draft mà em đo trực tiếp từ log server chỉ có ~30-35% thôi, nên chi phí
verify cộng với forward pass của draft model vượt quá phần compute tiết kiệm được,
nhất là khi target model vốn đã decode khá nhanh rồi (single request, chạy Metal). Đây
đúng kiểu tình huống mà CHALLENGES.md có cảnh báo trước — "production engine tắt spec
decode khi batch size cao" — chỉ khác là ở máy em, ngưỡng đó bị vượt ngay từ batch
size 1 luôn.

C8 (semantic cache) thì cho em một insight hoàn toàn khác, về *chất lượng* chứ không
phải tốc độ: cache dùng embedding lấy từ chat model pooled (không phải embedding model
chuyên dụng) nên vừa bỏ sót true paraphrase thật (#3, #6 chỉ đạt sim 0.72-0.80, dưới
ngưỡng 0.85), vừa false-hit một topic hoàn toàn mới (#7, sim 0.85). Em thử nghĩ không
ra ngưỡng nào sửa được cả hai lỗi cùng lúc — và chính đó là bằng chứng cụ thể cho việc
tại sao semantic cache thật sự cần một embedding model chuyên dụng, chứ không thể tái
dùng chat model như vầy được.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Điều làm em bất ngờ nhất là speculative decoding (C1) lại làm decode **chậm hơn**
(0.70x), chứ không nhanh hơn — ngược hẳn với con số 3-6.5x của EAGLE-3 mà deck nêu.
Bất ngờ thứ hai là build từ source mà em kỳ vọng "tối ưu riêng cho đúng CPU của mình"
lại thua bản prebuilt (0.82x), vì hoá ra trên Apple Silicon thì NEON đã là cái sàn bắt
buộc rồi, chứ không phải thứ mà `-DGGML_NATIVE` có thể mở khoá thêm được. Vui nhất là
cả hai lần này đều rơi vào tình huống giống nhau: "làm nhiều hơn" (thêm draft model,
compile riêng cho CPU) lại hoá ra chậm hơn "làm ít hơn" (cứ dùng baseline, prebuilt
binary sẵn có).

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
