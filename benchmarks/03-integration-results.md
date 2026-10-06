# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 7328.6 | 7328.7 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5515.5 | 5515.6 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 5659.6 | 5659.7 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **6167.9** · total **6168.0**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Thành phần trong lần chạy | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Chạy local, chưa nối hạ tầng cloud/IaC | Stub / chưa tích hợp |
| N17 Data pipeline | Dữ liệu mẫu TOY_DOCS, chưa nối pipeline N17 | Stub |
| N18 Lakehouse | Danh sách tài liệu trong bộ nhớ, chưa nối lakehouse | Stub |
| N19 Vector + features | Keyword overlap fallback, không có embedding hoặc vector index | Stub |
| N20 Serving | Gọi HTTP đến llama-server local, trả lời đủ 3 query | Real |

Latency trung bình: embed 0,0 ms, retrieve 0,1 ms, llm 6.167,9 ms, total 6.168,0 ms. Embed gần 0 vì không chạy mô hình embedding, không phải embedding thật xử lý tức thời. Em thấy LLM chiếm gần 100%, phù hợp với retrieval rất nhỏ trên dữ liệu mẫu. Để giảm tổng latency 2 lần, em ưu tiên tối ưu LLM: kiểm chứng thread count dưới cùng workload, giảm context hoặc giới hạn output nếu vẫn đủ chất lượng; tối ưu retrieval gần như không giúp trong lần chạy này.
