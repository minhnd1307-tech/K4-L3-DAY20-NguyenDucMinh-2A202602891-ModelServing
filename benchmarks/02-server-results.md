# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=4` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 11 | 0.19 | 29000 | 47000 | 47000 | 5.6 | 0.0% |
| 50 | 12 | 0.29 | 29000 | 42000 | 42000 | 8.2 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.55x** (31% of linear) |
| P95 latency | **0.89x** |
| Effective concurrency at 50 users | 8.2 vs `--parallel 4` slots (occupancy/slot ratio 2.04) |

**Saturated.** Throughput delivered only 1.55x for 5x the offered load, and effective concurrency (8.2) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

P95 decreased (0.89x) while throughput rose 1.55x. This alone does not establish spare capacity: the slot and queue metrics show contention, and the completed-request samples are small and differ in prompt mix.

> **Small sample.** Only 11 requests completed in the
> shorter run, so these percentiles are indicative rather than solid. Note also that
> locust averages only *completed* requests: when the run ends with requests still
> queued, effective concurrency is an **under**-estimate. Trust the throughput-scaling
> row over the concurrency row here, and run longer (`-t 3m`) if you want firmer numbers.

## Your reading

Từ 10 lên 50 users, RPS tăng từ khoảng 0,19 lên 0,29 (1,55 lần theo số chưa làm tròn), thấp hơn mức tăng 5 lần của số users. Effective concurrency ở 50 users khoảng 8,2, vượt 4 slot; metrics ghi nhận peak 3,82/4 slot bận và tối đa 46 request chờ. Em thấy dấu hiệu bão hòa ở tải 50 users, nhưng hai mức tải chưa đủ xác định chính xác điểm bắt đầu.

P95 giảm từ 47 xuống 42 giây (0,89 lần), không tăng như kỳ vọng. Mỗi lần chỉ hoàn tất 11–12 request; load-10 có 2 request long-rag, còn load-50 chỉ có request short hoàn tất. Khác biệt mẫu và các request chưa hoàn tất có thể ảnh hưởng percentile. Vì vậy, em không dùng P95 giảm để kết luận server còn dư công suất; hàng chờ đã được metrics xác nhận.

Em chọn SLO P95 ≤ 30 giây: cả hai lần đều chưa đạt. Chưa thể tính chính xác goodput@30s từ các percentile tổng hợp. Em sẽ kiểm tra server thực sự dùng 1 thread và đối chiếu với 4 thread dưới cùng tải, vì CP2 cho thấy 1 thread nhanh hơn trong llama-bench nhưng chưa chứng minh lợi ích khi phục vụ nhiều request. Em ưu tiên kiểm chứng knob này trước khi tăng parallel, vì tăng slot có thể làm tăng áp lực bộ nhớ mà không tăng tốc độ xử lý.

Em đã chạy lại load-10 vì CSV lần đầu chưa khớp bảng cuối ở terminal; báo cáo này dùng CSV của lần chạy lại, đã khớp ảnh 04-locust-10.png.
