# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=8` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 74 | 1.28 | 6700 | 11000 | 12000 | 8.5 | 0.0% |
| 50 | 91 | 1.54 | 30000 | 36000 | 37000 | 36.4 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.20x** (24% of linear) |
| P95 latency | **3.27x** |
| Effective concurrency at 50 users | 36.4 vs `--parallel 4` slots (occupancy/slot ratio 9.11) |

**Saturated.** Throughput delivered only 1.20x for 5x the offered load, and effective concurrency (36.4) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.20x while P95 moved 3.27x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

- **Điểm bão hòa (Saturation point):** Server bão hòa rõ rệt khi tải vượt quá 10–15 users và bão hòa nặng ở 50 users.
- **Bằng chứng số liệu:**
  1. *Throughput sụp đổ tốc độ tăng:* Khi tải tăng 5× (10 lên 50 users), RPS chỉ tăng nhẹ từ 1.28 lên 1.54 (tăng 1.20×, chỉ đạt 24% mức tăng tuyến tính).
  2. *Độ trễ bùng nổ:* P95 tăng vọt **3.27×** (từ 11,000 ms lên 36,000 ms), P50 tăng từ 6,700 ms lên 30,000 ms.
  3. *Tích lũy hàng đợi:* Effective concurrency đạt 36.4 requests (tỷ lệ chiếm dụng 9.11× so với 4 slot tính toán vật lý), và `requests_deferred` chạm đỉnh 46. Toàn bộ độ trễ tăng thêm ở 50 users là queue time (thời gian nằm chờ trong hàng đợi) chứ không phải compute time.
- **Knob ưu tiên điều chỉnh để nâng Goodput@SLO:**
  + Nếu đặt mục tiêu SLO là $P95 \le 12$s, ở mức 50 users goodput thực tế giảm về gần 0 do phần lớn request vi phạm SLO vì xếp hàng.
  + **Knob cần đổi trước:** **Admission Control / Concurrency Limiting (giới hạn hàng đợi và shed load)** kết hợp cấu hình `max_deferred_requests`. Khi số request in-flight vượt quá 4-8, server cần chủ động trả về HTTP 429/503 để từ chối tải dư thừa thay vì nhận toàn bộ vào queue, từ đó bảo vệ độ trễ P95 ở ngưỡng 11s và giữ Goodput tối đa ở mức ~1.3 - 1.5 RPS. Trên CPU-only, việc tăng `--parallel` bị chặn bởi memory bandwidth và RAM, do đó admission control là vũ khí hiệu quả nhất.
