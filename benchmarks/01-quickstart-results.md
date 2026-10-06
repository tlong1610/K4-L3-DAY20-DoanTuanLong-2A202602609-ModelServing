# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Windows-AMD64` · llama.cpp `b10488`
Settings: `threads=6` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4379 | 218 / 498 | 12.9 / 13.3 | 1021 / 1325 / 1325 | 77.8 |
| UD-Q2_K_XL | 2.24 | 4707 | 219 / 335 | 13.1 / 13.2 | 1036 / 1159 / 1159 | 76.6 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `UD-Q4_K_XL` decode within 2% of each other here, for 0.73 GB difference on disk.

## Your observation

**Trên máy này, 2-bit không đáng dùng.** Bench chạy với `ngl=99`, tức toàn bộ model nằm trên
RTX 3050 (4 GB VRAM).

- **Kích thước:** Q2 nhỏ hơn 0.73 GB (2.24 so với 2.97 GB, **−25%**).
- **Tốc độ:** Q2 **không nhanh hơn**. Decode 76.6 so với 77.8 tok/s (chậm hơn ~1.5%). TPOT P50
  13.1 so với 12.9 ms. TTFT P50 gần như bằng nhau (219 so với 218 ms). Load chậm hơn một chút
  (4707 so với 4379 ms).
- **Chất lượng:** hỏi cùng 3 câu trên cả hai bản (temperature 0, qua `/v1/chat/completions`):
  - Cả hai đều tính đúng giờ tàu (14:35 + 2h50 = 17:25). Nhưng Q4 làm đúng yêu cầu
    "trả lời trong một câu", còn Q2 bỏ qua yêu cầu định dạng và trả về một danh sách dài.
  - Câu "vì sao decode bị giới hạn bởi memory bandwidth": Q4 nêu đúng cơ chế (sinh từng
    token một nên mỗi bước phải đọc lại toàn bộ weight; giới hạn là tốc độ đọc bộ nhớ, không
    phải FLOPs). Q2 nêu một lý do sai, "context window lớn".
  - Câu tiếng Việt về continuous batching: cả hai trôi chảy và gần tương đương.

Vì sao Q2 nhỏ hơn 25% mà không nhanh hơn? Lý thuyết "decode bị giới hạn bởi băng thông" dự
đoán ít byte hơn thì nhanh hơn, nên kết quả này cần giải thích. Mình nghĩ có ba nguyên nhân:

1. Các bản Unsloth "UD" (dynamic) giữ những tensor nhạy cảm ở số bit cao. Một phần chênh lệch
   kích thước có thể nằm ở các bảng embedding: mỗi token chỉ tra một dòng trong bảng, không
   phải đọc cả bảng. Vì vậy số byte thật sự đọc mỗi token giảm ít hơn 25%.
2. Kernel giải nén Q2_K tốn nhiều phép tính hơn trên mỗi byte so với Q4_K.
3. Ở khoảng 13 ms/token trên GPU, một phần chi phí cố định (khởi chạy kernel, sampling,
   stream HTTP từng token) không phụ thuộc vào số bit.

Q4 vừa VRAM 4 GB nên Q2 không mang lại lợi ích gì, chỉ làm giảm chất lượng. Q2 chỉ đáng dùng
khi Q4 không vừa VRAM hoặc RAM, vì khi đó lựa chọn thật là Q2 trên GPU so với Q4 bị đẩy một
phần ra CPU.
