# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 6.8 | 38% |
| 3 | 14.4 | 80% |
| 6 | 18.0 | 100% |
| 12 | 15.9 | 88% |
| 24 | 12.1 | 67% |

**Best**: `-t 6` at 18.0 tok/s
**Slowest tested**: `-t 1` at 6.8 tok/s (2.64x spread)
**Against the physical-core default** (`-t 6`, 18.0 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=6 make bench
```

## Your explanation

*(Lượt này chạy bằng `LAB_N_GPU_LAYERS=0 make tune` để ép chạy hoàn toàn trên CPU. Lượt mặc định
với GPU nằm ở `01-tuning-tg128.md`.)*

**Knee ở `-t 6`, đúng bằng số nhân vật lý.** Từ 1 lên 3 lên 6 thread, tốc độ tăng 6.8 → 14.4
→ 18.0 tok/s. Tăng gần tuyến tính lúc đầu, rồi chậm lại: từ 3 lên 6 thread chỉ thêm 25%, trong
khi số nhân tăng gấp đôi. Đây là dấu hiệu bắt đầu chạm giới hạn băng thông RAM.

Cụ thể, mỗi token decode phải đọc toàn bộ các weight đang dùng từ RAM. Khi đủ nhân để đẩy đầy
băng thông của 2 kênh DDR4, thêm thread cũng không có thêm byte/s nào để dùng. Ước tính thô:
18 tok/s × khoảng 2 GB đọc mỗi token ≈ 36 GB/s. Con số này đã gần trần lý thuyết ~51 GB/s của
DDR4-3200 hai kênh. (Đây là ước tính: mình không đo cấu hình RAM, và số byte thật sự đọc mỗi
token của Gemma 4 E2B nhỏ hơn kích thước file.)

**Vượt quá 6 thread thì chậm đi:** 12 thread còn 15.9 tok/s (−12%), 24 thread còn 12.1 tok/s
(−33%).
- Ở 12 thread, hai luồng hyperthread chia nhau một nhân vật lý, tức dùng chung đơn vị
  AVX-512/FMA và cache L1/L2. Phần tính toán không tăng, nhưng các thread phải đồng bộ với
  nhau ở barrier sau mỗi phép nhân ma trận, nên thread chậm nhất kéo cả nhóm chậm theo.
- Ở 24 thread (gấp đôi số nhân logic), hệ điều hành phải liên tục đổi thread ra vào trên cùng
  một nhân. Mỗi lần đổi làm mất cache, và barrier phải chờ những thread đang bị tạm dừng.

Kết luận: với decode trên CPU, đặt `-t` bằng số nhân vật lý. Mặc định của lab (6) đã là điểm
tốt nhất.
