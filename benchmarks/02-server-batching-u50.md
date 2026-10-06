# 02 - Continuous batching under load (u50)

Host `Windows-AMD64` · `--parallel 4` · 15 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.96 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 45 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 16481 |

Highest sampled value was **3.96 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

**Batch width cao nhất đo được: 3.96 / 4 slot (99%).** `make metrics` chạy chồng thời gian với
`make load-50`, nên giá trị gần bằng `--parallel` này là bằng chứng continuous batching thật sự
hoạt động: mỗi bước decode xử lý gần 4 request cùng lúc. Nếu server phục vụ từng request một,
giá trị sẽ khoảng 1, như lúc chạy `make smoke` (`n_busy_slots_per_decode = 1.00`).

Con số này **không mâu thuẫn** với effective concurrency 40.6 trong `02-server-results.md`, vì
hai số đo hai thứ khác nhau:
- **3.96** là số request *đang được decode* (bị chặn trên bởi 4 slot).
- **40.6** (Little's Law: RPS × latency trung bình) là số request *đang nằm trong hệ thống*,
  gồm cả những request đang xếp hàng.

Phần chênh lệch, khoảng 40.6 − 4 ≈ 36 request, là hàng đợi. `requests_deferred` đạt đỉnh 45
cũng xác nhận điều này: lúc cao điểm có tới 45 request đang chờ slot.

Để biết mức sử dụng slot, mình tin gauge của server. Để biết người dùng phải chờ bao lâu, mình
tin con số từ Little's Law. Gộp hai số lại: slot đã đầy 99%, và phần lớn latency ở 50 user là
thời gian chờ trong hàng, không phải thời gian tính toán.
