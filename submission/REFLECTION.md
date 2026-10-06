# Reflection — Day 20 Lab

**Họ tên:** Nguyễn Đức Minh

**MSSV:** 2A202602891

**Cohort:** 4

**Lớp:** Track 2

**Ngày submit dự kiến:** 2026-10-06

## 1. Hardware & runtime

- **OS:** Windows 10 (AMD64), theo hardware.json.
- **CPU:** Intel Core i7-8565U @ 1.80 GHz; 4 core vật lý, 8 luồng.
- **CPU extensions:** Probe chưa ghi thông tin này.
- **RAM:** 15,9 GB.
- **Accelerator:** NVIDIA GeForce MX130, 2.048 MiB VRAM; probe cũng phát hiện Vulkan.
- **Runtime:** llama.cpp b10488, asset llama-b10488-bin-win-cuda-12.4-x64.zip.
- **Model:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`).
- **Quantization:** UD-Q4_K_XL và UD-Q2_K_XL.
- **Chạy ở đâu:** Laptop local của em, không dùng cloud.

**Setup story:** Em thêm UTF-8 BOM vào lab.ps1 để PowerShell 5.1 đọc đúng mã hóa. Launcher serve.py được sửa để dùng subprocess trên Windows, tránh tách sai đường dẫn có khoảng trắng. Em tải runtime prebuilt và hai bản model, không compile. Em chạy lại load-10 vì CSV ban đầu chưa khớp bảng cuối ở terminal.

## 2. Đo lường

Benchmark dùng 4 thread, ngl=99, ctx=2048, max_tokens=64; bỏ warm-up, mỗi quant hoàn tất 10/10 request.

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 45496 | 1839 / 1876 | 165.6 / 166.1 | 12272 / 12327 / 12327 | 6.0 |
| UD-Q2_K_XL | 2.24 | 6653 | 1949 / 1987 | 181.4 / 181.5 | 13380 / 13422 / 13422 | 5.5 |

**Quan sát:** Bản 2-bit nhỏ hơn 24,6% nhưng decode chậm hơn khoảng 1,09 lần. Cả hai trả lời đúng 440.000 đồng cho cùng bài toán giảm giá/VAT khi max_tokens=512. Em chọn 4-bit vì nhanh hơn trong lần đo này; một câu hỏi chưa đủ kết luận chất lượng tổng thể tương đương.

## 3. Serving under load

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.19 | 29000 | 47000 | 47000 | 5.6 | 0.0% |
| 50 | 0.29 | 29000 | 42000 | 42000 | 8.2 | 0.0% |

- **Số users tăng:** 5 lần; **RPS tăng:** 1,55 lần theo số chưa làm tròn.
- **Tỷ lệ P95:** 0,89 lần, tức giảm khoảng 10,6%.
- **Effective concurrency ở 50 users:** 8,2 so với 4 slot; bao gồm hàng chờ.
- **Peak n_busy_slots_per_decode:** 3,82/4 slot; requests_deferred tối đa 46.
- **Smoke:** Completion thật, tokens_predicted_total=20, khác 0.

**Saturation reading:** Em thấy dấu hiệu bão hòa ở 50 users: RPS tăng ít, slot gần đầy, có 46 request chờ. P95 giảm chưa chứng minh còn dư công suất vì chỉ có 11–12 mẫu, mix prompt khác nhau. SLO P95 ≤ 30 giây chưa đạt. Em ưu tiên kiểm chứng 1 thread dưới cùng tải; CP2 chưa chứng minh speedup serving.

Bảng dùng load-10 chạy lại, đã khớp ảnh; không sửa tay số liệu. Header load-report ghi threads=4 từ môi trường tạo báo cáo, chưa xác nhận số thread của server đang chạy.

## 4. Integration

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | Chạy local, chưa nối cloud/IaC | Stub / chưa tích hợp |
| N17 Data pipeline | Dữ liệu mẫu TOY_DOCS | Stub |
| N18 Lakehouse | Danh sách tài liệu trong bộ nhớ | Stub |
| N19 Vector + features | Keyword overlap, không dùng embedding/vector index | Stub |
| N20 Serving | llama-server qua HTTP, đủ 3 query | Real |

**Latency split trung bình:** embed **0,0 ms**, retrieve **0,1 ms**, llm **6.167,9 ms**, total **6.168,0 ms**. LLM chiếm gần **100%**; embed gần 0 vì không chạy mô hình embedding.

**Reflection:** Em thấy LLM là bottleneck, đúng kỳ vọng vì retrieval chỉ dùng dữ liệu mẫu nhỏ. Để giảm latency 2 lần, em ưu tiên tối ưu LLM: kiểm chứng thread count, giảm context hoặc output nếu vẫn đủ chất lượng. Tối ưu retrieval gần như không giúp trong lần chạy này.

## 5. The single change that mattered most

**Change:** Giảm thread count từ mặc định 4 xuống 1, trên cùng model UD-Q4_K_XL với ngl=99, đo bằng llama-bench tg128.

```text
before:  4 threads, 5.9 tok/s
after:  1 thread, 9.3 tok/s
speedup: 1.59x
```

Em thấy 1 thread đạt cao nhất; 2 thread giảm xuống 5,8 tok/s, còn 8 và 16 thread cùng đạt 6,7 tok/s. Kết quả khác kỳ vọng peak quanh 4 core vật lý. Khi cấu hình yêu cầu GPU offload, phần việc CPU có thể nhỏ; thêm thread làm tăng chi phí điều phối và đồng bộ mà không tăng tốc phần GPU. Tranh chấp cache hoặc băng thông cũng có thể ảnh hưởng.

Đây là giả thuyết cơ chế, chưa được phép đo này xác nhận. Em chọn 1 thread theo kết quả sweep, nhưng speedup 1,59 lần chỉ áp dụng cho llama-bench tg128 đã đo, không khẳng định toàn bộ API/RAG nhanh hơn tương ứng. Cần đo lặp lại và đối chiếu cùng workload để xác nhận độ ổn định.

## 6. Bonus

Em chưa làm bonus; tập trung hoàn thành base track.

## 7. Điều làm em ngạc nhiên nhất

Bản 2-bit nhỏ hơn nhưng chậm hơn, và 1 thread lại nhanh hơn 4 thread. Em thấy cần đo trên máy thực tế thay vì suy ra tốc độ từ số bit hoặc số luồng.

## 8. Self-check trước khi push

- [x] Có hardware.json và models/active.json.
- [x] Có các báo cáo benchmark, tuning, load, batching và integration; đã điền nhận xét.
- [x] Có CSV load-10 và load-50.
- [x] Có đủ ảnh: 01-hardware-probe, 02-bench, 03a-serve + 03b-smoke, 04-locust-10, 05-locust-50; thêm 07-metrics và 08-pipeline.
- [x] Tên thư mục repo đúng: K4-L3-DAY20-NguyenDucMinh-2A202602891-ModelServing.
- [ ] Commit đầy đủ file nộp bài và các sửa lỗi launcher.
- [ ] Chạy verify đạt exit 0 sau khi đưa đủ file vào Git.
- [ ] Xác nhận repo GitHub public, push và nộp URL vào LMS trước deadline.
- [ ] Kiểm tra commit không chứa model weights, runtime, venv hoặc secrets.

## 9. Khai báo sử dụng AI

Em dùng ChatGPT/Codex để đọc hướng dẫn lab, giải thích chỉ số, sửa lỗi PowerShell/launcher, đối chiếu số liệu, lưu ảnh và hỗ trợ viết nhận xét từ kết quả thực tế. Em tự chạy các phép đo trên laptop; AI không tạo số liệu hoặc ảnh bằng chứng.
