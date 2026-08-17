# Project Docs

Bộ ghi chú lý thuyết + kinh nghiệm áp dụng thực tế trong dự án. README này là mục lục — bấm vào từng mục để đi thẳng đến nội dung.

## 📚 Danh sách tài liệu

### 1. [Caching Patterns — Lý thuyết & Áp dụng](CACHING-PATTERNS.md)

Các pattern caching với Redis: lý thuyết kèm ví dụ thực tế trong dự án.

- [1. Nền tảng: Cache-Aside (Lazy Loading)](CACHING-PATTERNS.md#1-nền-tảng-cache-aside-lazy-loading)
- [2. Chọn TTL: câu hỏi đầu tiên của mọi cache](CACHING-PATTERNS.md#2-chọn-ttl-câu-hỏi-đầu-tiên-của-mọi-cache)
- [3. Version-Keyed Invalidation (Generation-based)](CACHING-PATTERNS.md#3-version-keyed-invalidation-generation-based)
- [4. Cache Stampede & Single-Flight Lock](CACHING-PATTERNS.md#4-cache-stampede--single-flight-lock)
- [5. Lazy Invalidation vs Eager Refresh](CACHING-PATTERNS.md#5-lazy-invalidation-vs-eager-refresh)
- [6. Kỷ luật invalidation: blast radius & thứ tự](CACHING-PATTERNS.md#6-kỷ-luật-invalidation-blast-radius--thứ-tự)
- [7. Biết cái gì KHÔNG cache](CACHING-PATTERNS.md#7-biết-cái-gì-không-cache)
- [8. Checklist áp dụng cho dự án khác](CACHING-PATTERNS.md#8-checklist-áp-dụng-cho-dự-án-khác)
- [9. Các chiến lược caching khác (tham khảo nhanh)](CACHING-PATTERNS.md#9-các-chiến-lược-caching-khác-tham-khảo-nhanh)
  - Read-Through · Write-Through · Write-Behind · Refresh-Ahead · Stale-While-Revalidate · Negative Caching · L1/L2 · Probabilistic Early Expiration

### 2. [NestJS DI Scope & Memory Leak](NESTJS-SCOPE.md)

Lý thuyết ngắn về scope của provider trong NestJS và các vector gây memory leak.

- [1. Ba scope của provider](NESTJS-SCOPE.md#1-ba-scope-của-provider)
- [2. Hai luật quan trọng của Request scope](NESTJS-SCOPE.md#2-hai-luật-quan-trọng-của-request-scope)
- [3. Hợp đồng của Singleton: stateless service](NESTJS-SCOPE.md#3-hợp-đồng-của-singleton-stateless-service)
- [4. Memory leak trong thế giới singleton — 4 vector](NESTJS-SCOPE.md#4-memory-leak-trong-thế-giới-singleton--4-vector)
- [5. Cần context theo request mà không muốn trả giá Request scope?](NESTJS-SCOPE.md#5-cần-context-theo-request-mà-không-muốn-trả-giá-request-scope)
- [Tóm tắt phỏng vấn (30 giây)](NESTJS-SCOPE.md#tóm-tắt-phỏng-vấn-30-giây)

### 3. [Transactional Outbox & Change Data Capture (CDC)](TRANSACTIONAL-OUTBOX-AND-CDC.md)

Hai kỹ thuật đảm bảo data consistency khi một service phải vừa ghi DB vừa phát event, kèm ví dụ thực tế từ module `venue-registry` của nollie-api.

- [1. Bài toán gốc: Dual-Write Problem](TRANSACTIONAL-OUTBOX-AND-CDC.md#1-bài-toán-gốc-dual-write-problem)
- [2. Transactional Outbox Pattern](TRANSACTIONAL-OUTBOX-AND-CDC.md#2-transactional-outbox-pattern)
- [3. Change Data Capture (CDC)](TRANSACTIONAL-OUTBOX-AND-CDC.md#3-change-data-capture-cdc)
- [4. So sánh nhanh](TRANSACTIONAL-OUTBOX-AND-CDC.md#4-so-sánh-nhanh)
- [5. Đọc thêm trong repo](TRANSACTIONAL-OUTBOX-AND-CDC.md#5-đọc-thêm-trong-repo)

### 4. [Mã hóa dữ liệu PII & Search trên dữ liệu mã hóa](PII-ENCRYPTION-THEORY.md)

Lý thuyết về bảo vệ dữ liệu cá nhân (PII): tại sao phải mã hóa, lựa chọn thuật toán, AES-256-CBC, và search trên dữ liệu đã mã hóa với blind index.

- [1. Vấn đề](PII-ENCRYPTION-THEORY.md#1-vấn-đề)
- [2. Có những lựa chọn thuật toán nào?](PII-ENCRYPTION-THEORY.md#2-có-những-lựa-chọn-thuật-toán-nào)
- [3. Chi tiết AES-256-CBC](PII-ENCRYPTION-THEORY.md#3-chi-tiết-aes-256-cbc)
- [4. Search trên dữ liệu mã hóa — Blind Index](PII-ENCRYPTION-THEORY.md#4-search-trên-dữ-liệu-mã-hóa--blind-index)
- [5. Tổng kết](PII-ENCRYPTION-THEORY.md#5-tổng-kết)

### 5. [Concurrency Control — Xử lý Race Condition & Đảm bảo Toàn vẹn Dữ liệu](RACE-CONDITION-HANDBOOK.md)

Các cơ chế kiểm soát đồng thời (locking, optimistic/pessimistic, v.v.) và cách chọn; ví dụ từ bài toán booking, case study nollie-api.

- [1. Nguyên lý cốt lõi](RACE-CONDITION-HANDBOOK.md#1-nguyên-lý-cốt-lõi)
- [2. Tổng quan các cơ chế kiểm soát đồng thời](RACE-CONDITION-HANDBOOK.md#2-tổng-quan-các-cơ-chế-kiểm-soát-đồng-thời)
- [3. Phân tích chi tiết từng cơ chế](RACE-CONDITION-HANDBOOK.md#3-phân-tích-chi-tiết-từng-cơ-chế)
- [4. Tiêu chí lựa chọn cơ chế](RACE-CONDITION-HANDBOOK.md#4-tiêu-chí-lựa-chọn-cơ-chế)
- [5. Các lỗi thiết kế thường gặp](RACE-CONDITION-HANDBOOK.md#5-các-lỗi-thiết-kế-thường-gặp)
- [6. Case study: booking engine của nollie-api](RACE-CONDITION-HANDBOOK.md#6-case-study-booking-engine-của-nollie-api)
- [7. Mẫu trả lời phỏng vấn — scenario "N client tranh 1 bàn cuối"](RACE-CONDITION-HANDBOOK.md#7-mẫu-trả-lời-phỏng-vấn--scenario-n-client-tranh-1-bàn-cuối)
- [8. Checklist khi review code](RACE-CONDITION-HANDBOOK.md#8-checklist-khi-review-code)

### 6. [Concurrency Control — Lý thuyết và cách nollie-api áp dụng](CONCURRENCY-CONTROL.md)

Pessimistic vs optimistic locking trong PostgreSQL (FOR UPDATE, advisory lock, version column), map vào codebase nollie-api và các bẫy thực tế.

- [1. Vấn đề gốc: race condition](CONCURRENCY-CONTROL.md#1-vấn-đề-gốc-race-condition)
- [2. Lý thuyết](CONCURRENCY-CONTROL.md#2-lý-thuyết)
- [3. nollie-api đang dùng gì](CONCURRENCY-CONTROL.md#3-nollie-api-đang-dùng-gì)
- [4. Bẫy thực tế (đã gặp hoặc dễ gặp trong repo này)](CONCURRENCY-CONTROL.md#4-bẫy-thực-tế-đã-gặp-hoặc-dễ-gặp-trong-repo-này)
- [5. Checklist khi viết code mới có shared write](CONCURRENCY-CONTROL.md#5-checklist-khi-viết-code-mới-có-shared-write)

### 7. [Kiến trúc Streaming Fan-out cho đồng bộ dữ liệu khối lượng lớn](STREAMING-SYNC-ARCHITECTURE.md)

Mẫu thiết kế nhập dữ liệu lớn từ API bên thứ ba có phân trang: fan-out nhiều worker song song + hàng đợi trung gian + batch writer, đảm bảo nhanh, chịu lỗi và idempotent.

- [1. Ý tưởng cốt lõi](STREAMING-SYNC-ARCHITECTURE.md#1-ý-tưởng-cốt-lõi)
- [2. Mô hình tổng quát trong một câu](STREAMING-SYNC-ARCHITECTURE.md#2-mô-hình-tổng-quát-trong-một-câu)
- [3. Các kỹ thuật thành phần đáng tái sử dụng](STREAMING-SYNC-ARCHITECTURE.md#3-các-kỹ-thuật-thành-phần-đáng-tái-sử-dụng)
- [4. So sánh với mô hình tuần tự page-by-page](STREAMING-SYNC-ARCHITECTURE.md#4-so-sánh-với-mô-hình-tuần-tự-page-by-page)
- [5. Cái giá phải trả — nhìn thẳng vào trade-off](STREAMING-SYNC-ARCHITECTURE.md#5-cái-giá-phải-trả--nhìn-thẳng-vào-trade-off)
