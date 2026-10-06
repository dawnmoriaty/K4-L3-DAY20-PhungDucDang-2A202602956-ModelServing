# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 3391.6 | 3391.8 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 1500.9 | 1501.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 1110.3 | 1110.4 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **2000.9** · total **2001.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it explicitly accounts for **SLOs (Service Level Objectives)** and **TPOT (Throughput at Saturation)**.

Here is the breakdown of why this makes Goodput superior:

1.  **SLO Compliance**: The context states that Goodput counts only requests per second that met the **TTFT** (Total Throughput at Failure) and **TPOT

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation** in GPU memory.

By storing the Key-Value (KV) cache in non-contiguous pages, it avoids the wasteful fragmentation that would occur if all KV data were packed tightly into a single contiguous block of memory. This allows the engine to utilize more of the available GPU memory for other tasks, such as the prefilling of the attention mech

**When does splitting prefill and decode help?**

> Based on the context provided, splitting prefill and decode helps when **both operations are compute-bound**.

The context explicitly states that prefill is compute-bound and decode is memory-bandwidth-bound. By splitting them, the system can utilize different hardware resources (compute vs. memory) for each phase, which is beneficial when both are resource-intensive.


## Which N16-N19 pieces are real

- **Phân loại hiện trạng các module N16–N19:**
  + **N16 (Cloud Infra / K8s):** Stubbed (chạy trực tiếp trên Linux host/bare-metal, không deploy cluster K8s).
  + **N17 (Data Pipeline / Kafka CDC):** Stubbed (sử dụng context tĩnh có sẵn thay vì luồng streaming Kafka thật).
  + **N18 (Lakehouse / Delta Lake / Iceberg):** Stubbed (lưu trữ tệp cục bộ trên filesystem thay vì bảng định dạng ACID Lakehouse).
  + **N19 (Vector & Feature Store):** Real (tích hợp qua backend keyword overlap fallback trực tiếp gọi đến inference server).
  + **N20 (Serving):** Real (`llama-server` chạy song song 4 slot continuous batching thật).

- **Đánh giá bottleneck (Dominant stage):**
  + Hoàn toàn đúng như kỳ vọng: Tầng **`llm` chiếm trọn 100% thời gian** (trung bình 2,000.9 ms trên tổng 2,001.1 ms; retrieval chỉ tốn 0.1 ms và embed 0.0 ms). Trong kiến trúc phục vụ mô hình ngôn ngữ trên CPU, việc sinh tự hồi quy (autoregressive decode) và prefill hàng trăm token đòi hỏi duyệt toàn bộ trọng số mô hình qua bộ nhớ ở mỗi token, áp đảo hoàn toàn thao tác tra cứu vector/keyword tính bằng microsecond.

- **Chiến lược giảm 2× độ trễ:**
  + Theo Định luật Amdahl, tối ưu tầng retrieval hay embed là vô nghĩa vì tỷ trọng quá nhỏ. Bắt buộc phải **tấn công trực diện vào tầng `llm`**:
    1. Áp dụng **Prompt Caching / Prefix Caching** để tái sử dụng KV cache của phần context/system prompt chung, giảm thiểu thời gian prefill.
    2. Sử dụng **Speculative Decoding** với draft model nhỏ hơn (ví dụ 0.1B-0.3B) để sinh nhiều token mỗi bước bộ nhớ.
    3. Giới hạn `max_tokens` chặt chẽ theo từng tác vụ trả lời để cắt ngắn giai đoạn autoregressive decode tốn kém.
