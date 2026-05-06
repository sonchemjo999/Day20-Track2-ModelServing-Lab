# Reflection - Lab 20 (Personal Report)

> Day 20 - Model Serving & Inference Optimization. Báo cáo cá nhân dựa trên artifact đã có trong repo và screenshot.

---

**Họ Tên:** Đào Hồng Sơn  
**MSSV:** 2A202600462  
**Ngày submit:** 06/05/2026

---

## 1. Hardware spec (từ `00-setup/detect-hardware.py`)

- **OS:** Linux 5.15.0-163-generic (x86_64)
- **CPU:** Intel(R) Xeon(R) Platinum 8269CY CPU @ 2.50GHz
- **Cores:** 4 physical / 4 logical
- **CPU extensions:** AVX-512 available
- **RAM:** 7.8 GB
- **Accelerator:** CPU only, không có discrete accelerator
- **llama.cpp backend đã chọn:** CPU (AVX/NEON tuning), `n_gpu_layers=0`
- **Recommended model tier:** TinyLlama-1.1B (Q4_K_M)

**Setup story** (<= 80 chữ):  
Máy chạy trong môi trường Linux CPU-only nên mình chọn TinyLlama-1.1B để vừa RAM 7.8 GB. Không có CUDA/Metal/Vulkan, vì vậy benchmark dùng `n_threads=4`, `n_ctx=2048`, `n_batch=512`, `n_gpu_layers=0`. Model chính là Q4_K_M, model so sánh là Q2_K.

---

## 2. Track 01 - Quickstart numbers (từ `benchmarks/01-quickstart-results.md`)

| Model | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode rate (tok/s) |
|---|--:|--:|--:|--:|--:|
| tinyllama-1.1b-chat-v1.0.Q4_K_M.gguf | 1281 | 221 / 266 | 50.6 / 51.6 | 3384 / 3449 / 3468 | 19.8 |
| tinyllama-1.1b-chat-v1.0.Q2_K.gguf | 236 | 378 / 497 | 48.0 / 49.3 | 3377 / 3523 / 3545 | 20.8 |

**Một quan sát** (<= 50 chữ):  
Q2_K load nhanh hơn và decode nhanh hơn nhẹ, nhưng E2E gần như ngang Q4_K_M. Trên máy này Q4_K_M đáng chọn hơn vì mất rất ít latency nhưng chất lượng text thường ổn định hơn.

---

## 3. Track 02 - llama-server load test

Locust screenshot hiển thị response-time percentile, không tách riêng TTFB. Cột TTFB bên dưới vì vậy ghi `N/A`; P95/P99 lấy từ dòng Aggregated trong screenshot.

| Concurrency | Total RPS | TTFB P50 (ms) | E2E P95 (ms) | E2E P99 (ms) | Failures |
|--:|--:|--:|--:|--:|--:|
| 10 | ~0.10 | N/A (E2E P50 28000) | 50000 | 50000 | 0 observed |
| 50 | ~0.13 | N/A (E2E P50 37000) | 52000 | 52000 | 0 observed |

**KV-cache observation** (từ `record-metrics.py`):  
Peak `llamacpp:kv_cache_usage_ratio` trong CSV hiện có = `0.00`. Dòng CSV được ghi lúc server idle (`requests_processing=0`, `requests_deferred=0`), nên nó chỉ xác nhận metrics endpoint scrape được, chưa phản ánh peak KV-cache khi concurrency 50 đang chạy. Khi tải thật, cần chạy `record-metrics.py` song song với Locust để lấy peak đúng.

---

## 4. Track 03 - Milestone integration

- **N16 (Cloud/IaC):** stub: localhost only, llama-server chạy tại `http://localhost:8080/v1`
- **N17 (Data pipeline):** stub: batch/in-process Python flow trong `pipeline.py`
- **N18 (Lakehouse):** stub: không dùng lakehouse thật, context nằm trong Python list
- **N19 (Vector + Feature Store):** stub: `TOY_DOCS` + keyword-overlap retrieval, chưa dùng vector DB/feature store

**Nơi tốn nhiều ms nhất** trong pipeline (đo bằng `time.perf_counter` trong `pipeline.py`):

- embed: 0.0 ms (chưa có embedder, retrieval đang keyword-based)
- retrieve: ~0.1 ms expected trên `TOY_DOCS`
- llama-server: chưa có log timing pipeline committed; đây sẽ là phần lớn nhất vì mỗi request phải sinh token trên CPU

**Reflection** (<= 60 chữ):  
Bottleneck nằm ở llama-server, khớp với kỳ vọng vì embed/retrieve đang stub rất nhỏ còn decode trên CPU tốn hàng chục giây dưới Locust. Nếu thay bằng vector DB thật, retrieval sẽ tăng nhưng vẫn khó vượt chi phí sinh token của TinyLlama CPU-only.

---

## 5. Bonus - The single change that mattered most

**Change:** Chọn Q2_K thay cho Q4_K_M khi ưu tiên latency/load-time trên CPU-only RAM 7.8 GB.

**Before vs after** (từ quickstart quant comparison):

```text
before: Q4_K_M load=1281 ms, TPOT P50=50.6 ms, decode=19.8 tok/s, E2E P50=3384 ms
after:  Q2_K   load=236 ms,  TPOT P50=48.0 ms, decode=20.8 tok/s, E2E P50=3377 ms
speedup: ~1.05x decode, ~5.4x model-load speedup
```

**Tại sao nó work:**  
Trên CPU-only, model nhỏ hơn giúp giảm lượng weight phải đọc từ RAM và cache mỗi lần load/prefill. Q2_K nên load nhanh hơn rất rõ, vì file quant nhẹ hơn và ít dữ liệu phải map/đọc vào memory. Decode cũng nhanh hơn một chút vì mỗi token cần ít memory bandwidth hơn.

Tuy nhiên speedup decode chỉ khoảng 5%, không phải 2-4x. Điều này cho thấy workload sau khi model đã load không chỉ bị giới hạn bởi kích thước weight; overhead Python/llama.cpp, prompt length, cache, và CPU scheduling cũng đóng vai trò lớn. Vì E2E gần như ngang nhau, mình sẽ vẫn chọn Q4_K_M nếu cần chất lượng trả lời, chỉ dùng Q2_K khi máy rất thiếu RAM hoặc cần khởi động model nhanh.

---

## 6. (Optional) Điều ngạc nhiên nhất

Điều bất ngờ là Q2_K load nhanh hơn rất nhiều nhưng E2E P50 gần như không đổi. Nếu chỉ nhìn file size thì dễ kỳ vọng Q2_K nhanh hơn nhiều, nhưng số liệu cho thấy decode trên CPU có nhiều giới hạn khác ngoài quantization.

---

## 7. Self-graded checklist

- [x] `hardware.json` đã commit
- [x] `models/active.json` đã commit
- [x] `benchmarks/01-quickstart-results.md` đã commit
- [x] `benchmarks/02-server-metrics.csv` đã commit
- [ ] `benchmarks/bonus-*.md` đã commit (chưa có bonus sweep file)
- [ ] Ít nhất 6 screenshots trong `submission/screenshots/` (hiện có 5 screenshots)
- [ ] `make verify` exit 0 (chưa pass vì thiếu GGUF file referenced trong `models/active.json` và thiếu screenshot thứ 6)
- [ ] Repo trên GitHub ở chế độ public
- [ ] Đã paste public repo URL vào VinUni LMS

---

**Quan trọng:** repo phải public đến khi điểm được công bố. Nếu private, grader không xem được -> 0 điểm.
