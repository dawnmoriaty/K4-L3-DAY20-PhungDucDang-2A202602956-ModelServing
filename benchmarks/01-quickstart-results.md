# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=8` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 3162 | 132 / 188 | 14.8 / 15.4 | 1045 / 1137 / 1137 | 67.8 |
| UD-Q2_K_XL | 0.39 | 2027 | 179 / 244 | 14.8 / 15.6 | 1110 / 1226 / 1226 | 67.8 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `Q4_K_M` decode within 2% of each other here, for 0.11 GB difference on disk.

## Your observation

1. **Tốc độ Decode (TPOT - Memory bandwidth bound)**:
   - Cả hai bản `Q4_K_M` và `UD-Q2_K_XL` đều cho tốc độ decode giống hệt nhau là **14.8 ms/token** (tương đương **67.8 tok/s** ở P50), và chỉ chênh lệch 1.3% ở P95 (15.4 ms vs 15.6 ms).
   - **Cơ chế**: Do mô hình `Qwen3.5 0.8B` có kích thước rất nhỏ (~0.5 GB), toàn bộ weights nằm gọn trong RAM và tận dụng tối đa cache L3 của CPU AMD Ryzen AI 7 350. Vì vậy, hệ thống không bị nghẽn băng thông bộ nhớ (memory bandwidth) ở kích thước mô hình này. Việc giảm dung lượng từ 4-bit xuống 2-bit không giúp tăng tốc độ decode.

2. **Độ trễ phản hồi ban đầu (TTFT - Compute/FLOPs bound)**:
   - Bản 2-bit (`UD-Q2_K_XL`) có TTFT P50 chậm hơn bản 4-bit (`Q4_K_M`) tới **35.6%** (179 ms so với 132 ms), và P95 chậm hơn **29.8%** (244 ms so với 188 ms).
   - **Cơ chế**: Giai đoạn prefill phụ thuộc hoàn toàn vào năng lực tính toán của CPU (compute-bound). Định dạng Unsloth Dynamic Q2_K_XL sử dụng cấu trúc lượng tử hoá phi tuyến phức tạp; CPU tốn thêm nhiều chu kỳ tính toán để giải nén (dequantize) các block trọng số này so với định dạng Q4_K_M chuẩn vốn được tập lệnh AVX-512 xử lý vector hoá cực kỳ tối ưu.

3. **Thời gian tải mô hình (Load time)**:
   - Bản 2-bit tải nhanh hơn 35.9% (2,027 ms so với 3,162 ms) nhờ file nhỏ hơn 22% (0.39 GB vs 0.50 GB).

4. **Kết luận đánh đổi (Trade-off & Usefulness)**:
   - Bản 2-bit chỉ tiết kiệm được 0.11 GB dung lượng lưu trữ/RAM. Đổi lại, TTFT bị chậm hơn đáng kể (tăng thêm ~47 ms ở P50) mà tốc độ decode hoàn toàn không tăng, đồng thời khả năng biểu đạt ngôn ngữ của 2-bit ở model 0.8B bị suy giảm rõ rệt.
   - Do đó, trên máy tính này với 14.9 GB RAM, **bản 4-bit (`Q4_K_M`) là lựa chọn vượt trội hoàn toàn về trải nghiệm người dùng và chất lượng sinh văn bản**.
