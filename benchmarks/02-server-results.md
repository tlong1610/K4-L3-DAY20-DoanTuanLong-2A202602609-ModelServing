# 02 - Serve: load test + saturation reading

Host `Windows-AMD64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=6` ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 145 | 2.49 | 2900 | 4900 | 7000 | 7.7 | 0.0% |
| 50 | 151 | 2.59 | 17000 | 19000 | 21000 | 40.6 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.04x** (21% of linear) |
| P95 latency | **3.88x** |
| Effective concurrency at 50 users | 40.6 vs `--parallel 4` slots (occupancy/slot ratio 10.15) |

**Saturated.** Throughput delivered only 1.04x for 5x the offered load, and effective concurrency (40.6) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.04x while P95 moved 3.88x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Server đã bão hoà ngay từ 10 user.** Trần throughput khoảng **2.5 RPS**.

Con số thuyết phục nhất là throughput: tải tăng 5× nhưng RPS chỉ đi từ 2.49 lên 2.59 (1.04×).
Thêm 40 user không tạo ra thêm công việc được hoàn thành.

Ngay ở 10 user, effective concurrency đã là 7.7, lớn hơn 4 slot, tức đã có request phải xếp
hàng. Tách latency bằng Little's Law:
- Khi 4 slot luôn bận, thời gian phục vụ mỗi request ≈ 4 slot / 2.59 RPS ≈ **1.5 s**.
- Ở 50 user, latency trung bình W = 40.6 / 2.59 ≈ 15.7 s. Vậy khoảng **14 s (~90%) là thời gian
  chờ trong hàng**, chỉ khoảng 1.5 s là thời gian tính toán.
- Ở 10 user, W ≈ 3.1 s, gồm khoảng 1.6 s tính toán và khoảng 1.5 s chờ.

Đó là lý do P95 tăng 3.88× (4.9 s → 19 s) trong khi throughput gần như không đổi.

Một lưu ý về cách đo: locust gọi `http://localhost:8080`. Trên máy này, mỗi kết nối mới tới
`localhost` mất khoảng 2.2 s, vì Windows thử IPv6 `::1` trước rồi mới chuyển sang IPv4, trong
khi server chỉ nghe trên `127.0.0.1` (đo riêng: 2246 ms so với 172 ms). Locust giữ kết nối sống
cho mỗi user, nên chỉ request đầu tiên của mỗi user bị cộng thêm khoảng 2.2 s phía client. Lỗi
này làm P99 lệch lên một chút, nhưng không đổi kết luận (17 s so với 2.2 s).

**SLO mình chọn:** P95 end-to-end ≤ 5 s cho câu trả lời 48–96 token.
- Ở 10 user: P95 = 4.9 s, vừa đạt SLO.
- Ở 50 user: P50 đã 17 s, nên goodput@SLO gần bằng 0, dù throughput vẫn 2.59 RPS.

**Knob mình sẽ đổi trước: `--parallel` 4 → 8, kèm `LAB_N_CTX=4096`** để mỗi slot vẫn có 512 token
context. Lý do:
- Gauge cho thấy slot là thứ đang nghẽn: `n_busy_slots_per_decode` = 3.96/4 và
  `requests_deferred` = 45.
- Decode trên GPU bị giới hạn bởi băng thông bộ nhớ. Mỗi bước decode đọc weight một lần, dùng
  chung cho mọi slot trong batch. Thêm slot nên gần như cộng thêm token mỗi bước mà không tốn
  thêm thời gian đọc weight, cho tới khi chạm giới hạn tính toán hoặc hết VRAM cho KV cache
  (4 GB VRAM mà model đã chiếm 2.97 GB).

Đổi thread hay quantization sẽ không giúp: `tune` đã cho thấy `-t` không có tác dụng khi chạy
trên GPU, và bench cho thấy Q2 không decode nhanh hơn Q4.

Ở 50 user, thêm slot vẫn không đủ để đạt SLO. Cần thêm admission control: giới hạn độ dài hàng
đợi hoặc trả lỗi 429 sớm, để những request được nhận vẫn nằm trong SLO.
