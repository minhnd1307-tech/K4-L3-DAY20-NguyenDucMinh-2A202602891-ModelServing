# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.82 of 4 slots (95%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 1277 |

Highest sampled value was **3.82 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Em ghi nhận peak n_busy_slots_per_decode là 3,82/4 slot (khoảng 95%), cùng 4 request đang xử lý và tối đa 46 request chờ. Gauge này là trung bình số slot bận mỗi bước decode, không phải batch width tức thời. Giá trị gần 4 cho thấy continuous batching hoạt động dưới tải 50 users.

Effective concurrency trong load-report là khoảng 8,2, lớn hơn 4 slot vì tính cả request đang chờ; do đó không mâu thuẫn với gauge 3,82. Em dùng metrics để đánh giá số slot bận, còn Little's Law để ước lượng request đang xử lý hoặc xếp hàng. Load test chỉ hoàn tất 12 request nên ước lượng này còn hạn chế.
