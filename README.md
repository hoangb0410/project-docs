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
