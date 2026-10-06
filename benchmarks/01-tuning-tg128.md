# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Windows-AMD64` · llama.cpp `b10488`
CPU: **6 physical · 12 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 81.7 | 100% |
| 3 | 81.7 | 100% |
| 6 | 81.7 | 100% |
| 12 | 81.7 | 100% |
| 24 | 81.6 | 100% |

**Best**: `-t 3` at 81.7 tok/s
**Slowest tested**: `-t 24` at 81.6 tok/s (1.00x spread)
**Against the physical-core default** (`-t 6`, 81.7 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=3 make bench
```

## Your explanation

**Đường cong phẳng hoàn toàn:** 81.6–81.7 tok/s từ 1 đến 24 thread, **không có knee**. Kết quả
này khác với hình dạng mà deck dự đoán, và lý do nằm ở `ngl=99`. Toàn bộ layer của model đã
nằm trên RTX 3050, nên decode chạy trên GPU và bị giới hạn bởi băng thông GDDR6 của GPU, không
phải RAM của CPU.

Khi đó, các thread CPU chỉ làm việc điều phối: khởi chạy kernel CUDA, sampling, tokenize. Một
thread là đủ cho việc này, nên `-t` không còn là một knob có tác dụng. Mấy "best" `-t 3` mà
script chọn chỉ là nhiễu ở chữ số thập phân thứ hai (81.7 so với 81.6).

Để thấy được knee, mình chạy lại với `LAB_N_GPU_LAYERS=0` (file `01-tuning-tg128-cpu-only.md`).
Ở đó đỉnh nằm đúng ở 6 nhân vật lý, 18.0 tok/s. So hai lượt với nhau: **chuyển decode sang GPU
nhanh hơn 81.7 / 18.0 = 4.5× so với cấu hình CPU tốt nhất.** Đây là thay đổi lớn nhất trên máy
này, phân tích ở REFLECTION §5.
