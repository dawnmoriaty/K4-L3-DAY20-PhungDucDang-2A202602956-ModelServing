# Bonus - Context-length sweep (prefill cost)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=8` `ngl=0` · RAM 14.9 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 267.0 | 958.8 | 1.00x |
| 1024 | 220.9 | 4635.0 | 1.21x |
| 2048 | 250.4 | 8177.3 | 1.07x |
| 4096 | 241.3 | 16974.0 | 1.11x |
| 8192 | 222.0 | 36899.2 | 1.20x |

At 8192 tokens, prefill costs **36899 ms** --
1.20x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

- **Ngưỡng Prefill áp đảo End-to-End Latency:** 
  Ở prompt ngắn (256 tokens), prefill chỉ mất 958.8 ms (< 1s). Nhưng khi prompt chạm ngưỡng **2048 tokens**, thời gian prefill vọt lên **8,177 ms (~8.2s)**, và tại **8192 tokens** bùng nổ lên tới **36,899 ms (~36.9s)**. Ở mốc này, TTFT hoàn toàn nuốt chửng toàn bộ trải nghiệm người dùng, lớn hơn gấp nhiều lần thời gian sinh văn bản (decode ~1-2s cho 50-100 tokens).

- **Quan sát đường cong Attention (Quadratic bend):**
  Từ 256 đến 2048 tokens, prefill tăng xấp xỉ tuyến tính vì chi phí per-layer MLP/Projection O(N) áp đảo. Nhưng khi sequence length tiến đến 8192 tokens, thành phần tính toán ma trận Attention $O(N^2)$ bắt đầu lộ rõ, khiến chi phí prefill cao gấp **1.20×** so với mức dự đoán tuyến tính.

- **Ý nghĩa thực tiễn đối với RAG Pipeline:**
  Không bao giờ được lạm dụng việc nhồi nhét nhiều context chunks chỉ vì mô hình có cửa sổ ngữ cảnh rộng. Mỗi 512 tokens tài liệu retrieve thêm vào prompt sẽ phạt trực tiếp ~2.1 giây vào TTFT trước khi token đầu tiên kịp hiển thị. Một RAG pipeline tối ưu trên CPU chỉ nên duy trì ngân sách prompt từ **1024 – 2048 tokens** (tương đương 2–3 chunks chắt lọc kỹ càng) để giữ TTFT dưới ngưỡng 5–8 giây.
