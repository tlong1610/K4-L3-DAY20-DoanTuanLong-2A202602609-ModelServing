# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Đoàn Tuấn Long
**MSSV:** 2A202602609
**Cohort:** A20-K4
**Ngày submit:** 2026-10-06

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Windows 11 Pro (AMD64)
- **CPU:** Intel Core i5-11400H @ 2.70GHz (Tiger Lake-H)
- **Cores:** 6 physical / 12 logical
- **CPU extensions:** AVX2, AVX-512
- **RAM:** 15.8 GB
- **Accelerator:** NVIDIA GeForce RTX 3050 Laptop GPU (4 GB VRAM), CUDA (driver 610.62); Vulkan cũng có
- **llama.cpp asset đã tải:** `llama-b10488-bin-win-cuda-12.4-x64.zip` + `cudart-llama-bin-win-cuda-12.4-x64.zip`
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi, không dùng cloud. Mọi lần đo đều offload toàn bộ model lên GPU
(`ngl=99`), trừ lượt `tune` chỉ chạy CPU mà tôi cố ý ép `ngl=0`.

**Setup story** (≤ 80 chữ): Mạng trường làm tải runtime bị treo, nên tôi tải zip bằng
`curl --retry -C -` rồi giải nén. `.\lab.ps1` lỗi cú pháp trên PowerShell 5.1 (file UTF-8 không
BOM), nên tôi gọi thẳng các script Python. Report ghi bằng cp1252 nên tôi chuyển sang UTF-8. Ở
web UI, mỗi slot chỉ có 512 token (2048 / 4 slot), phải tăng `LAB_N_CTX`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4379 | 218 / 498 | 12.9 / 13.3 | 1021 / 1325 / 1325 | 77.8 |
| UD-Q2_K_XL | 2.24 | 4707 | 219 / 335 | 13.1 / 13.2 | 1036 / 1159 / 1159 | 76.6 |

**Quan sát** (≤ 60 chữ): 2-bit nhỏ hơn 25% nhưng **không nhanh hơn**: decode 76.6 so với 77.8
tok/s. Hỏi cùng 3 câu (temperature 0): cả hai tính đúng 17:25, nhưng Q2 bỏ qua yêu cầu "một câu"
và giải thích sai vì sao decode bị giới hạn bởi băng thông. Q4 vừa 4 GB VRAM, nên **2-bit không
đáng dùng** trên máy này.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 2.49 | 2900 | 4900 | 7000 | 7.7 | 0.0% |
| 50 | 2.59 | 17000 | 19000 | 21000 | 40.6 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.04×
- **P95 tăng:** 3.88×
- **Effective concurrency ở 50 users:** 40.6 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.96 / 4 slots (`requests_deferred` đạt đỉnh 45)

**Saturation reading** (≤ 80 chữ): Server bão hoà từ khoảng 10 user, trần khoảng 2.5 RPS: tải
tăng 5× mà RPS chỉ tăng 1.04×. Theo Little's Law, mỗi request được phục vụ trong khoảng
4 / 2.59 ≈ 1.5 s, trong khi latency trung bình ở 50 user là 15.7 s. Vậy khoảng 90% là thời gian
chờ trong hàng; số slot bận 3.96/4 và deferred = 45 xác nhận điều đó. Knob đầu tiên tôi đổi là
`--parallel` 8 (kèm ctx 4096): decode trên GPU đọc weight một lần cho cả batch, nên thêm slot gần
như không tốn thêm.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không dùng, chạy local trên laptop | stub |
| N17 Data pipeline | `TOY_DOCS` viết cứng trong `pipeline.py` | stub |
| N18 Lakehouse | list Python trong bộ nhớ, không có lakehouse | stub |
| N19 Vector + features | không có embedding, retrieve bằng keyword overlap | stub |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py --base-url http://127.0.0.1:8080`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 903.0 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): Bottleneck là llm, đúng như kỳ vọng. Điều bất ngờ là lần chạy mặc
định với `localhost` cho llm = 2729.5 ms: khoảng 2.2 s mỗi request là Windows thử IPv6 `::1`
trước rồi mới sang IPv4. Đổi sang `127.0.0.1` thì nhanh hơn 3×. Muốn giảm thêm 2×: giữ kết nối
sống (một `httpx.Client` dùng chung), stream token, giảm `max_tokens`, tận dụng prompt cache.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** chuyển decode từ CPU sang GPU: `-ngl 0` → `-ngl 99` (cả 35 layer lên RTX 3050).
Bên "before" dùng cấu hình CPU tốt nhất tìm được bằng thread sweep (`-t 6`). Đo bằng
`llama-bench tg128`, model UD-Q4_K_XL. Nguồn: `benchmarks/01-tuning-tg128-cpu-only.md` và
`benchmarks/01-tuning-tg128.md`.

```
before:  18.0 tok/s  (ngl=0, -t 6, tg128)
after:   81.7 tok/s  (ngl=99, -t bất kỳ, tg128)
speedup: 4.5×
```

**Tại sao nó work:**

Decode sinh từng token một. Mỗi token phải đọc lại toàn bộ weight đang dùng từ bộ nhớ, nhưng
với mỗi byte đọc vào chỉ làm vài phép nhân-cộng. Vì vậy tốc độ decode gần như bằng
**băng thông bộ nhớ ÷ số byte đọc mỗi token**, không phụ thuộc FLOPs. Trên CPU, weight nằm
trong DDR4 hai kênh, trần lý thuyết khoảng 51 GB/s nếu là DDR4-3200 (tôi không đo cấu hình RAM).
RTX 3050 Laptop dùng GDDR6 128-bit, khoảng 192–224 GB/s theo thông số NVIDIA, tức gấp
**khoảng 4×**. Speedup đo được là 4.5×, cùng bậc với tỉ lệ băng thông này. Nếu decode bị giới
hạn bởi tính toán, tỉ lệ sẽ khác hẳn.

Thread sweep cũng khớp với giải thích này:
- Trên CPU, tốc độ tăng 6.8 → 14.4 → 18.0 tok/s (1 → 3 → 6 thread) rồi giảm: 15.9 tok/s ở 12
  thread, 12.1 tok/s ở 24 thread. Knee nằm đúng ở 6 nhân vật lý, vì đến đó băng thông RAM đã
  đầy. Thêm hyperthread chỉ làm các thread tranh nhau đơn vị AVX-512 và cache L1/L2 của cùng một
  nhân. Còn ở 24 thread, hệ điều hành phải đổi thread ra vào liên tục, nên các barrier sau mỗi
  phép nhân ma trận phải chờ những thread đang bị tạm dừng.
- Khi `ngl=99`, đường cong **phẳng hoàn toàn** (81.6–81.7 tok/s từ 1 đến 24 thread). Đây là
  chỗ khác với kỳ vọng "knee ở số nhân vật lý" trong deck. Lý do: CPU không còn làm phép nhân ma
  trận nữa, chỉ khởi chạy kernel CUDA và sampling. Một thread là đủ, nên knob `-t` mất tác dụng.

Bài học: phải biết bottleneck nằm ở bộ nhớ nào rồi mới chọn knob. Trên CPU, knob đúng là `-t`
bằng số nhân vật lý. Trên GPU, knob đó không còn ý nghĩa.

---

## 6. Bonus  *(optional — tối đa 10 điểm)*

> Bỏ trống nếu không làm. Xem `docs/bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

Có hai điều:
- Q2 nhỏ hơn 25% mà không decode nhanh hơn chút nào trên GPU.
- Chi phí lớn nhất trong RAG pipeline ban đầu không phải LLM mà là việc phân giải `localhost`
  (khoảng 2.2 s mỗi kết nối, do Windows thử IPv6 trước IPv4). Một chi tiết hạ tầng nhỏ có thể
  làm latency đầu-cuối tăng gấp 3 lần.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [x] Repo tên đúng mẫu `K4-L3-DAY20-HoVaTen-MSSV-ModelServing` (xem `docs/SUBMISSION.md`)
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã push và paste public URL vào VinUni LMS **trước 23:59 (UTC+7) ngày làm lab**
- [x] **Không** commit `models/*.gguf`, `runtime/` hay `.env` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.

---

## 9. Khai báo sử dụng AI  *(xem `docs/RULES.md` §3)*

Tôi dùng **Claude Code (Claude Opus 5.5)** cho các việc sau:
- Đọc đề và lên kế hoạch.
- Xử lý lỗi setup: tải runtime bị treo, `lab.ps1` lỗi encoding, file report bị ghi bằng cp1252.
- Chạy các lệnh đo: tune, load test, metrics, pipeline, so chất lượng Q4/Q2 qua API.
- Phát hiện và đo lỗi chậm khi gọi `localhost`.
- Viết bản nháp các phần nhận xét dựa trên số liệu đo trên máy tôi.

Mọi con số đều do script của lab sinh ra trên máy này, không sửa tay. Screenshot do tôi tự chụp.
Tôi đã đọc lại và hiểu các lập luận trong §2–§5 và trong các file `benchmarks/*.md`.
