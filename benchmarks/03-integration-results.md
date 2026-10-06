# 03 - Integrate: RAG pipeline run

Host `Windows-AMD64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 1686.1 | 1686.2 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 502.4 | 502.4 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 520.6 | 520.7 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **903.0** · total **903.1**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | không dùng, chạy local trên laptop | stub |
| N17 Data pipeline | `TOY_DOCS` viết cứng trong `pipeline.py` | stub |
| N18 Lakehouse | không có, tài liệu là một list Python trong bộ nhớ | stub |
| N19 Vector + features | không có embedding (`embed = 0 ms`), retrieve bằng keyword overlap | stub |
| N20 Serving | `llama-server` (Gemma 4 E2B Q4, CUDA) | **real** |

**Stage chiếm nhiều thời gian nhất: llm (100%)**, đúng như kỳ vọng. Phần embed và retrieve trên
toy data chỉ mất chưa tới 0.1 ms.

Có một điều bất ngờ: **lần chạy đầu tiên với URL mặc định `http://localhost:8080` cho llm
trung bình 2729.5 ms**, trong khi server chỉ báo khoảng 0.5 s (prefill ~100–250 ms + decode
~300 ms). Khoảng 2.2 s còn lại là lỗi phân giải `localhost` trên Windows: máy thử IPv6 `::1`
trước, thất bại, rồi mới sang `127.0.0.1`. Mỗi lần `httpx.post` mở một kết nối mới nên lần nào
cũng chịu khoảng 2.2 s.

Chạy lại với `--base-url http://127.0.0.1:8080` (bảng ở trên), latency trung bình còn
**903 ms (giảm khoảng 3×)**. Query đầu vẫn mất 1686 ms, vì còn chi phí tạo HTTP client và
prefill lần đầu (352 ms). Hai query sau chỉ khoảng 500 ms, và prefill chỉ còn 5 token vì
llama-server tái sử dụng KV cache của phần prompt trùng với lần chạy trước (prompt caching).

**Muốn giảm latency thêm 2×, mình sẽ tấn công stage llm:**
1. Dùng một `httpx.Client` giữ kết nối sống thay vì mở kết nối mới mỗi request.
2. Stream token ra cho người dùng, để latency cảm nhận được là TTFT chứ không phải E2E.
3. Giới hạn `max_tokens`, vì decode khoảng 12–13 ms/token chiếm phần lớn thời gian của server.
4. Đặt phần context cố định ở đầu prompt để tận dụng prompt cache.

Với context RAG dài thật (N19 thật), prefill sẽ tăng, và khi đó prompt caching cùng chunked
prefill mới là thứ quan trọng.
