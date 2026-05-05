# Reflection — Lab 19

**Tên:** Đào Hồng Sơn
**MSSV:** 2A202600462 
**Path đã chạy:** _lite_

---

## Câu hỏi (≤ 200 chữ)

Trên golden set 50 queries, mode **hybrid** thắng trung bình (78.6% Precision@10) vì nó robust trên mọi loại query. Chi tiết theo loại:

| Loại query | n  | Keyword | Semantic | Hybrid |
|------------|----|---------|----------|--------|
| exact      | 15 | 96.7%   | 88.7%   | 96.7%  |
| paraphrase | 15 | 33.3%   | 24.0%   | 32.0%  |
| mixed      | 20 | 97.0%   | 98.5%   | 100.0% |

- **`exact`**: BM25 thắng vì query chứa từ kỹ thuật verbatim trong corpus. Hybrid ngang BM25 vì keyword signal đã đủ mạnh, vector không cải thiện thêm.
- **`paraphrase`**: Semantic thua vì model `bge-small-en-v1.5` (English-trained) không tốt trên Vietnamese paraphrases. Đổi sang `bge-m3` sẽ giúp semantic thắng loại này — đây là teaching moment: embedding model phải phù hợp ngôn ngữ corpus.
- **`mixed`**: **Hybrid thắng áp đảo 100%** vì user thật hiếm khi viết 100% exact hoặc 100% paraphrase. Hybrid kết hợp cả hai signals.

**Khi nào không dùng hybrid:**
- **Pure BM25** khi: corpus nhỏ (<10K docs), query luôn chứa exact terms, cần latency cực thấp (P50=1.8ms so với hybrid P50=14.9ms), hoặc infra không hỗ trợ embedding model.
- **Pure vector** khi: query luôn paraphrase/ngữ cảnh mới, không có keyword terms trong corpus, hoặc cần multi-modal search (image+text) mà vector DB hỗ trợ native.

---

## Điều ngạc nhiên nhất khi làm lab này

Điều ngạc nhiên nhất là **pure semantic (vector) thua cả BM25** trên paraphrase queries (24% vs 33%) — ngược với kỳ vọng từ slide deck rằng vector search thắng trên semantic similarity. Nguyên nhân không phải ở thuật toán mà ở **embedding model mismatch**: `bge-small-en-v1.5` train trên tiếng Anh, corpus là tiếng Việt, nên embeddings không capture được semantic similarity đúng. Đây là bài học thực tế: embedding model phải match ngôn ngữ + domain của corpus, không phải cứ dùng vector search là "AI" hơn.

---

## Bonus challenge

- [ ] Đã làm bonus (xem `bonus/`)
- [ ] Pair work với: _<tên đồng đội nếu có>_

---

## Tổng kết deliverables đã hoàn thành

### NB1 — Embeddings & Vector Indexing
- Load 1000 docs từ `corpus_vn.jsonl` (10 topics × 100 docs)
- Embed bằng `BAAI/bge-small-en-v1.5` → 384-dim vectors
- Index trong Qdrant in-memory, query cosine similarity

### NB2 — Hybrid Search (BM25 + Vector + RRF)
- Build BM25 index (`rank_bm25`) + vector index (Qdrant) trên 1000 docs
- Implement RRF fusion: `score(d) = Σ 1/(k + rank)`, k=60
- Kết quả: Hybrid 78.6% > Keyword 77.8% > Semantic 73.2%
- Hybrid thắng tuyệt đối trên `mixed` queries (100%)

### NB3 — FastAPI `/search` Endpoint + Latency Benchmark
- API chạy FastAPI + uvicorn, endpoint `GET /search?q=...&mode=...`
- Server ready: `{'ready': True, 'n_docs': 1000}`
- Single query latency: 10.9ms
- Latency benchmark (100 queries × 3 modes):

| Mode     | P50    | P95    | P99    |
|----------|--------|--------|--------|
| keyword  | 1.8ms  | 4.6ms  | 12.8ms |
| semantic | 11.0ms | 25.3ms | 40.0ms |
| hybrid   | 14.9ms | 31.5ms | 46.3ms |

- **PASS**: Hybrid P99 server-side = 46.3ms < 50ms rubric threshold

### NB4 — Feast Feature Store (3 Feature Views)
- Sinh 3 Parquet files:
  - `user_profile.parquet` (2.8 KB, 100 users)
  - `item_popularity.parquet` (9.7 KB, 1000 items)
  - `query_velocity.parquet` (2.3 KB, 100 users)
- `feast apply`: Registered 3 feature views + 2 entities (user, item)
- `feast materialize-incremental`: Materialized 3 feature views vào SQLite online store
- Online lookup P99 = 5.05ms — **PASS** (< 10ms rubric threshold)
- PIT join demo: `get_historical_features` trả về feature value tại timestamp của event, không dùng giá trị tương lai (no data leakage)
