# README_submission.md

## Câu hỏi bắt buộc (LAB.md mục 5.2)

**1. Layer quan trọng nhất:** `long_term` — quyết định 4/11 case (E02, E03, E08, E09), 20/56 điểm auto, nhiều hơn mọi layer khác; xử lý cross-session recall, open-loop, recency/conflict và user isolation cùng lúc.

**2. Trade-off Context Block (Zep) vs Redis+Qdrant:** Context Block tự trích xuất/tóm tắt/xếp hạng relevance, không cần tự thiết kế schema hay similarity, nhưng managed/black-box, phụ thuộc độ trễ mạng (long_term ~1-1.2s). Redis là KV rẻ, nhanh, kiểm soát hoàn toàn nhưng không tự trích fact. Qdrant search chủ động nhưng phải tự embed/quản lý index, không có provenance/graph built-in như Zep.

**3. Guardrail chống memory poisoning:** (a) opt-in consent (`data/consent.json`) trước khi ghi durable memory; (b) `heartbeat.py` chỉ dedupe/mark-stale/tạo recap, **không** tự thêm instruction/quyền mới; (c) durable write giữ provenance (nguồn, timestamp) để audit/rollback; (d) high-impact preference change cần review trước khi ghi.

## Phân tích benchmark

**1. Layer hit rate thấp nhất:** Lần chạy cuối cả 4 layer đạt 100% (11/11). Có 1 lần `E09 (long_term)` fail do 404 tạm thời từ Zep API, pass ngay khi retry — long_term có độ trễ cao/biến động nhất (900-1200ms so với episodic/semantic ~200-500ms), là layer "mong manh" nhất về vận hành dù evidence đạt 100%.

**2. Query nhiều token nhất:** E02 (long_term, ngôn ngữ ưu tiên của Minh) — 882 token, vì Context Block trả cả user summary gồm fact không liên quan.

**3. E07 (mixed) cần layer nào:** `long_term` (Python preference) + `semantic` (`Idempotency-Key` từ payment KB dùng chung). Thiếu một trong hai sẽ fail.

**4. Token reduction:** No-memory giảm token 81.8% nhưng hit rate chỉ 18.2% (không retrieve gì). Memory-enabled giảm 14.2% token, hit rate 100%. Reduction chỉ có ý nghĩa cùng hit rate — bỏ hết context là "rẻ nhưng sai".

## E08 (recency) & E10 (compaction)

**E08:** Stage 1, Minh ưu tiên Python cho dự án cá nhân ORCHID-27. Stage 3, thêm dự án công ty BLUEBIRD-42, backend bắt buộc TypeScript/NestJS — tách rõ scope theo project, không ghi đè Python. Context Block giữ đúng update mới nhất theo từng scope.

**E10:** Buffer giữ toàn bộ message, token tăng tuyến tính, không co lại. Sliding nén history cũ thành `SESSION_SUMMARY`, tách riêng `DURABLE_NOTES` cho constraint quan trọng (`REVIEW-DEADLINE-1600`); giảm `max_recent_messages` 6→4, raw turn cũ bị evict khỏi `RECENT_TURNS` nhưng deadline vẫn còn trong durable note.
