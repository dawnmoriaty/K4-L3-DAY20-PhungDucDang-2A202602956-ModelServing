# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Phùng Đức Đăng
**MSSV:** 2A202602956
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Fedora Linux 40 (x86_64)
- **CPU:** AMD Ryzen AI 7 350 w/ Radeon 860M
- **Cores:** 8 physical / 16 logical
- **CPU extensions:** AVX2, AVX-512
- **RAM:** 14.9 GB
- **Accelerator:** CPU only
- **llama.cpp asset đã tải:** llama-b10488-bin-linux-avx512-x64.tar.gz (b10488)
- **Model đã dùng:** Qwen3.5 0.8B (`LAB_MODEL=qwen35-0.8b`)
- **Quantization:** Q4_K_M + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): Máy laptop chạy CPU AMD Zen 5 không có GPU rời nhưng hỗ trợ tập lệnh SIMD AVX-512. Em đã chọn runtime b10488 tối ưu AVX-512 và mô hình Qwen3.5 0.8B để đảm bảo model nằm gọn trong 14.9 GB RAM, tối đa hóa thông lượng bộ nhớ mà không cần GPU hay build từ source.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3162 | 132 / 188 | 14.8 / 15.4 | 1969 / 2095 / 2095 | 67.8 |
| UD-Q2_K_XL | 0.39 | 2027 | 179 / 244 | 14.8 / 15.6 | 1974 / 2189 / 2189 | 67.8 |

**Quan sát** (≤ 60 chữ): UD-Q2_K_XL giảm 22% dung lượng nhưng tốc độ decode ngang bằng (67.8 tok/s) vì model 0.8B nằm vừa L3 cache/RAM. Ngược lại, TTFT của bản 2-bit chậm hơn 35.6% do overhead giải nén dequantization phức tạp trên CPU. Đánh đổi chất lượng câu trả lời lấy dung lượng không xứng đáng ở model nhỏ này.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.28 | 6700 | 11000 | 12000 | 8.5 | 0.0% |
| 50 | 1.54 | 30000 | 36000 | 37000 | 36.4 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.20×
- **P95 tăng:** 3.27×
- **Effective concurrency ở 50 users:** 36.4 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.97 / 4 slots

**Saturation reading** (≤ 80 chữ): Server bão hòa ở 10–15 users. Bằng chứng: tải tăng 5× nhưng RPS chỉ tăng 1.20× (24% tuyến tính), còn P95 vọt 3.27× lên 36s. Thời gian tăng vọt là queue time vì `requests_deferred` chạm 46 và concurrency (36.4) áp đảo 4 slot. Để nâng Goodput@SLO, ưu tiên đổi knob admission control (giới hạn queue/shed load) trước vì tăng slot trên CPU sẽ bị nghẽn RAM bandwidth.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Bare-metal Linux host | stub |
| N17 Data pipeline | Static text corpus | stub |
| N18 Lakehouse | Local filesystem storage | stub |
| N19 Vector + features | Keyword overlap retrieval | real |
| N20 Serving | `llama-server` continuous batching | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.1 ms
- llm: 2000.9 ms
- **stage chiếm nhiều nhất:** llm (100.0% của total)

**Reflection** (≤ 60 chữ): Bottleneck nằm tuyệt đối ở tầng sinh LLM (2086 ms vs 0.1 ms retrieve), đúng với đặc thù CPU inference. Để giảm 2× latency, phải tấn công tầng LLM bằng prompt prefix caching và speculative decoding; tối ưu retrieval không đem lại ý nghĩa gì theo Định luật Amdahl.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Giảm số thread từ 16 (logical SMT threads) xuống 8 (physical CPU cores)

```
before:  38.9 tok/s
after:   67.2 tok/s
speedup: 1.73×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

Trên vi kiến trúc AMD Zen 5 (Ryzen AI 7 350), CPU gồm 8 physical cores với 16 logical threads thông qua SMT (Simultaneous Multithreading). Quá trình autoregressive decode của LLM đòi hỏi băng thông bộ nhớ cực lớn và tài nguyên thực thi SIMD AVX-512 liên tục. Khi chạy 16 threads, 2 logical threads trên cùng 1 core vật lý phải chia sẻ chung đơn vị phần cứng FPU AVX-512 và bộ đệm L1/L2 cache. Sự xung đột này gây nghẽn execution pipeline và cache thrashing nghiêm trọng, làm sụt giảm 42% hiệu năng decode.

Khi tinh chỉnh `-t` về đúng 8 threads (khớp số physical cores), mỗi luồng được cấp phát trọn vẹn một physical core, tận dụng tối đa băng thông L2/L3 cache và không phải chịu chi phí context switching hay tranh chấp FPU. Nhờ đó, thông lượng decode tăng vọt từ 38.9 lên 67.2 tok/s (tăng tốc 1.73×). Kết quả chứng minh rõ ràng: trong suy luận LLM trên CPU, số lượng thread vượt quá physical cores không giúp tăng tốc mà còn gây tổn hại hiệu năng nghiêm trọng do nút thắt băng thông bộ nhớ.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 sweep-ctx (Context-length sweep vs Prefill / TTFT)

**Numbers:**

```
before:  958.8 ms (256 prompt tokens)
after:   36899.2 ms (8192 prompt tokens)
speedup: 0.026×
```

**Điều này nói lên gì mà deck chưa nói:**

Deck lý thuyết thường nhấn mạnh tính chất "Attention là $O(N^2)$", nhưng đo đạc thực nghiệm trên vi kiến trúc CPU AVX-512 với mô hình Qwen3.5 0.8B cho thấy một bức tranh đa tầng hơn: ở dải prompt ngắn ($N < 2048$ tokens), chi phí tính toán các lớp MLP/Projection tuyến tính $O(N)$ vẫn chiếm ưu thế, duy trì tốc độ prefill ổn định ở mức ~240–267 tok/s. Chỉ khi ngữ cảnh vượt qua 4096 và chạm 8192 tokens, thành phần $O(N^2)$ của ma trận tương quan Self-Attention mới bộc lộ rõ rệt, khiến thời gian prefill bùng nổ lên tới 36.9 giây (vượt 1.20× so với mức tăng tuyến tính).

Hệ quả kiến trúc sâu sắc cho Model Serving: Việc nhồi nhét quá nhiều tài liệu retrieve vào RAG prompt biến một request từ trạng thái memory-bound (decode) sang trạng thái nặng nề compute-bound (prefill), đóng băng slot phục vụ trong hơn nửa phút. Đây chính là lý do thực tiễn bắt buộc các hệ thống production (vLLM/SGLang) phải triển khai kiến trúc **Disaggregated Prefill & Decode** (tách riêng server chuyên prefill và decode) hoặc **Chunked Prefill** để ngăn ngừa hiện tượng các prompt dài làm nghẽn (starve) các request decode đang chạy song song.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Sự sụt giảm hiệu năng mạnh mẽ khi dùng 16 threads SMT (từ 67.2 xuống 38.9 tok/s) và việc quantization 2-bit tốn chi phí giải nén trên CPU khiến TTFT chậm hơn 35.6% so với 4-bit.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [x] Repo GitHub ở chế độ **public**
- [x] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Sử dụng Antigravity IDE (Gemini 2.5) hỗ trợ phân tích vi kiến trúc phần cứng, chạy tự động kiểm thử benchmark và đối chiếu các chỉ số latency/throughput.
