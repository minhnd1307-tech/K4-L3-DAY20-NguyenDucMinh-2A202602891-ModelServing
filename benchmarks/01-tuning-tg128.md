# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **4 physical · 8 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 9.3 | 100% |
| 2 | 5.8 | 62% |
| 4 | 5.9 | 63% |
| 8 | 6.7 | 71% |
| 16 | 6.7 | 71% |

**Best**: `-t 1` at 9.3 tok/s
**Slowest tested**: `-t 2` at 5.8 tok/s (1.62x spread)
**Against the physical-core default** (`-t 4`, 5.9 tok/s): 1.59x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

Trong các cấu hình đã thử, 1 thread đạt cao nhất: 9,3 tok/s, so với 5,9 tok/s ở mặc định 4 thread, tương đương speedup 1,59 lần. Throughput giảm mạnh ở 2 thread, tăng nhẹ ở 4–8 thread rồi gần như không đổi ở 16 thread. Điểm tốt nhất nằm tại 1 thread, không phải 4 core vật lý như kỳ vọng cho CPU-only.

Lần đo dùng ngl=99, tức cấu hình yêu cầu GPU offload, nên đường cong không chỉ phản ánh khả năng xử lý của CPU. Khi phần việc trên CPU nhỏ, chi phí điều phối và đồng bộ giữa nhiều thread có thể lớn hơn lợi ích chạy song song; tranh chấp cache hoặc băng thông bộ nhớ cũng có thể ảnh hưởng. Đây là các giả thuyết, chưa được xác nhận bởi bảng này. Em chọn 1 thread cho cấu hình đã đo; cần đo lặp lại hoặc đối chiếu CPU-only để xác nhận kết quả và nguyên nhân.
