# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=4` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 45496 | 1839 / 1876 | 165.6 / 166.1 | 12272 / 12327 / 12327 | 6.0 |
| UD-Q2_K_XL | 2.24 | 6653 | 1949 / 1987 | 181.4 / 181.5 | 13380 / 13422 / 13422 | 5.5 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.09x SLOWER** than `UD-Q4_K_XL` here, despite being 0.73 GB smaller. That is a real result, not a mistake: fewer bits only buys speed when decode is limited by memory bandwidth. On a machine that is compute-limited instead — few cores, no GPU offload — the extra dequantization work of a heavily-quantized format can cost more than the bytes it saves. Say which case yours is.

## Your observation

Bản 2-bit nhỏ hơn 0,73 GB (24,6%), nhưng decode chậm hơn: 5,5 tok/s so với 6,0 tok/s của bản 4-bit. TTFT P50 và TPOT P50 của bản 4-bit cũng thấp hơn, lần lượt là 1.839 ms và 165,6 ms, so với 1.949 ms và 181,4 ms của bản 2-bit. Ít bit hơn không đảm bảo nhanh hơn; chi phí giải lượng tử có thể ảnh hưởng, nhưng số liệu này chưa đủ xác định nguyên nhân.

Với cùng câu hỏi giảm giá rồi tính VAT, cả hai bản đều trả lời đúng 440.000 đồng và kết thúc bằng stop khi dùng max_tokens = 512. Em chưa thấy giảm độ chính xác ở câu hỏi này, nhưng chưa thể kết luận cho mọi tác vụ. Em chọn bản 4-bit vì máy có đủ RAM và bản này nhanh hơn trong lần đo; bản 2-bit phù hợp khi cần tiết kiệm bộ nhớ.
