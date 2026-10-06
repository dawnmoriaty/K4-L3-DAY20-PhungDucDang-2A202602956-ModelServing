# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.97 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 16457 |

Highest sampled value was **3.97 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

- **Peak batch width:** Đạt **3.97 / 4 slots (99% công suất)**. Điều này chứng minh thuật toán Continuous Batching của `llama-server` đã hoạt động tối đa: gom 4 request đồng thời vào từng chu kỳ giải mã (decode step) mà không cần chờ đợi một batch cố định.
- **So sánh với Effective Concurrency (36.4 trong 02-server-results.md):** Hai con số có độ lệch lớn vì đo lường hai tầng khác nhau:
  + Gauge `n_busy_slots_per_decode` (3.97) đo mức độ chiếm dụng khe tính toán thực tế tại compute engine, bị giới hạn cứng bởi `--parallel 4`.
  + Effective Concurrency theo Định luật Little ($L = \lambda W = 36.4$) đo tổng số request đang tồn tại trong toàn bộ hệ thống (gồm cả request đang xử lý và request đang xếp hàng chờ). Metric Prometheus ghi nhận `requests_processing = 4` và `requests_deferred = 46` hoàn toàn khớp với hiện tượng này.
- **Độ tin cậy:** Tin cậy Prometheus gauge (3.97) để khẳng định năng lực song song hóa của engine, và tin cậy Little's Law (36.4) để giải thích sự bùng nổ hàng đợi khiến độ trễ P95 tăng vọt lên 36s.
