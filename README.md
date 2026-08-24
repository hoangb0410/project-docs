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

### 8. [Redis — Cấu trúc dữ liệu, lệnh và các pattern sử dụng thực tế](REDIS-GUIDE.md)

Kiến thức Redis nền tảng cho backend: các cấu trúc dữ liệu và lệnh chính, ba cấp độ atomic, Lua script, SCAN vs KEYS, và các pattern thực chiến (cache, lock, counter, rate limit, pub/sub, queue).

- [1. Redis là gì — mô hình trong một câu](REDIS-GUIDE.md#1-redis-là-gì--mô-hình-trong-một-câu)
- [2. Vòng đời key và quy ước đặt tên](REDIS-GUIDE.md#2-vòng-đời-key-và-quy-ước-đặt-tên)
- [3. Các cấu trúc dữ liệu và lệnh](REDIS-GUIDE.md#3-các-cấu-trúc-dữ-liệu-và-lệnh)
- [4. Ba cấp độ atomic — lệnh đơn, Lua script, pipeline](REDIS-GUIDE.md#4-ba-cấp-độ-atomic--lệnh-đơn-lua-script-pipeline)
- [5. Lua script trong Redis](REDIS-GUIDE.md#5-lua-script-trong-redis)
- [6. Duyệt keyspace: SCAN, không bao giờ KEYS](REDIS-GUIDE.md#6-duyệt-keyspace-scan-không-bao-giờ-keys)
- [7. Các pattern thực chiến](REDIS-GUIDE.md#7-các-pattern-thực-chiến)
- [8. Checklist thực dụng khi thêm một key Redis mới](REDIS-GUIDE.md#8-checklist-thực-dụng-khi-thêm-một-key-redis-mới)

### 9. [Stripe — Lý thuyết cốt lõi và tra cứu SDK](STRIPE-GUIDE.md)

Lý thuyết Stripe cho backend: nền tảng API, SDK stripe-node, các object thanh toán, Checkout Session, Connect, Billing/Subscription, webhook và các pattern thiết kế xuyên suốt.

- [1. Stripe là gì — mô hình trong một câu](STRIPE-GUIDE.md#1-stripe-là-gì--mô-hình-trong-một-câu)
- [2. Nền tảng API](STRIPE-GUIDE.md#2-nền-tảng-api)
- [3. SDK (stripe-node) — khởi tạo và quy ước gọi](STRIPE-GUIDE.md#3-sdk-stripe-node--khởi-tạo-và-quy-ước-gọi)
- [4. Các object thanh toán và method chính](STRIPE-GUIDE.md#4-các-object-thanh-toán-và-method-chính)
- [5. Checkout Session — trang thanh toán do Stripe host](STRIPE-GUIDE.md#5-checkout-session--trang-thanh-toán-do-stripe-host)
- [6. Stripe Connect — thu tiền hộ người khác](STRIPE-GUIDE.md#6-stripe-connect--thu-tiền-hộ-người-khác)
- [7. Billing — Subscription và họ hàng](STRIPE-GUIDE.md#7-billing--subscription-và-họ-hàng)
- [8. Webhook — kênh sự thật từ Stripe về](STRIPE-GUIDE.md#8-webhook--kênh-sự-thật-từ-stripe-về)
- [9. Các pattern thiết kế xuyên suốt](STRIPE-GUIDE.md#9-các-pattern-thiết-kế-xuyên-suốt)
- [10. Checklist khi thêm một thao tác Stripe mới](STRIPE-GUIDE.md#10-checklist-khi-thêm-một-thao-tác-stripe-mới)

### 10. [SendGrid — Lý thuyết cơ bản](SENDGRID-THEORY.md)

Gửi email qua SendGrid: khái niệm nền tảng, setup domain/sender, các SDK Node.js, batch email với `personalizations`, event webhook và chuyện IP cho subuser.

- [1. SendGrid là gì](SENDGRID-THEORY.md#1-sendgrid-là-gì)
- [2. Ưu điểm so với SMTP tự vận hành](SENDGRID-THEORY.md#2-ưu-điểm-so-với-smtp-tự-vận-hành)
- [3. Các khái niệm chính](SENDGRID-THEORY.md#3-các-khái-niệm-chính)
- [4. Setup (tổng quát)](SENDGRID-THEORY.md#4-setup-tổng-quát)
- [5. SDK Node.js](SENDGRID-THEORY.md#5-sdk-nodejs)
  - `@sendgrid/mail` · `@sendgrid/client` · `@sendgrid/eventwebhook`
- [6. Batch email — `personalizations`](SENDGRID-THEORY.md#6-batch-email--personalizations)
- [7. Event Webhook](SENDGRID-THEORY.md#7-event-webhook)
- [8. IP cho subuser & warm-up](SENDGRID-THEORY.md#8-ip-cho-subuser--warm-up)

### 11. [NestJS Fundamental](NESTJS-FUNDAMENTAL.md)

Ba nền tảng của NestJS: request đi qua những tầng nào, DI container hoạt động ra sao, và các design pattern mà framework dựng sẵn.

- [Request Lifecycle](NESTJS-FUNDAMENTAL.md#request-lifecycle)
  - Middleware · Guards · Interceptors (pre) · Pipes · Controller · Service · Interceptors (post) · Exception Filters
- [Dependency Injection](NESTJS-FUNDAMENTAL.md#dependency-injection)
  - Ba mảnh ghép · Các kiểu provider · Phạm vi module · Injection scope · Circular dependency · Vì sao DI làm test dễ hơn
- [Design Patterns trong NestJS](NESTJS-FUNDAMENTAL.md#design-patterns-trong-nestjs)
  - [Creational](NESTJS-FUNDAMENTAL.md#creational--tạo-ra-object) · [Structural](NESTJS-FUNDAMENTAL.md#structural--ghép-các-object-lại) · [Behavioral](NESTJS-FUNDAMENTAL.md#behavioral--điều-phối-luồng-và-giao-tiếp)

### 12. [Các mô hình giao tiếp Real-time](REALTIME-COMMUNICATION.md)

Bốn cách đưa dữ liệu từ server đến client: cơ chế, ví dụ Node.js tối giản, ví dụ thực tế trong nollie-api và tiêu chí chọn kỹ thuật.

- [1. Short Polling](REALTIME-COMMUNICATION.md#1-short-polling)
- [2. Long Polling](REALTIME-COMMUNICATION.md#2-long-polling)
- [3. Server-Sent Events (SSE)](REALTIME-COMMUNICATION.md#3-server-sent-events-sse)
- [4. WebSocket](REALTIME-COMMUNICATION.md#4-websocket)
  - [Giao thức (RFC 6455)](REALTIME-COMMUNICATION.md#41-giao-thức-websocket-rfc-6455) · [Thư viện lo tới đâu](REALTIME-COMMUNICATION.md#41b-thư-viện-lo-tới-đâu-bạn-lo-từ-đâu) · [socket.io](REALTIME-COMMUNICATION.md#42-socketio--wrapper-là-cách-hiểu-gần-đúng-nhưng-thiếu-một-ý-quan-trọng) · [Ứng dụng phải tự lo gì](REALTIME-COMMUNICATION.md#43-những-gì-ứng-dụng-phải-tự-lo)
- [Chọn kỹ thuật nào?](REALTIME-COMMUNICATION.md#chọn-kỹ-thuật-nào)
- [Bảng so sánh nhanh](REALTIME-COMMUNICATION.md#bảng-so-sánh-nhanh)
- [Bài học rút ra](REALTIME-COMMUNICATION.md#bài-học-rút-ra)

### 13. [Đăng nhập OAuth với Google và Facebook](OAUTH-FLOWS.md)

Ba luồng đăng nhập mạng xã hội — token flow, authorization code, authorization code + PKCE: cách hoạt động, chỗ dễ sai, so sánh và cách setup.

- [Bối cảnh: 3 bên và bài toán cần giải](OAUTH-FLOWS.md#bối-cảnh-3-bên-và-bài-toán-cần-giải)
- [1. Token flow](OAUTH-FLOWS.md#1-token-flow)
- [2. Authorization code flow](OAUTH-FLOWS.md#2-authorization-code-flow)
- [3. Authorization code + PKCE](OAUTH-FLOWS.md#3-authorization-code--pkce)
- [So sánh](OAUTH-FLOWS.md#so-sánh)
  - [Bảng đối chiếu](OAUTH-FLOWS.md#bảng-đối-chiếu) · [Chọn cái nào](OAUTH-FLOWS.md#chọn-cái-nào)
- [Setup nếu triển khai authorization code](OAUTH-FLOWS.md#setup-nếu-triển-khai-authorization-code)
  - [Google](OAUTH-FLOWS.md#google) · [Facebook](OAUTH-FLOWS.md#facebook) · [Backend](OAUTH-FLOWS.md#backend) · [Frontend](OAUTH-FLOWS.md#frontend)

### 14. [Availability & Table Allocator Engine — Tổng quan](AVAILABILITY-ALLOCATOR-OVERVIEW.md)

Một engine tính availability duy nhất cho widget/staff/agent: tách đọc khỏi ghi, chống trùng booking bằng ràng buộc Postgres (GiST exclusion constraint + advisory lock + row lock), cache version-keyed ở tầng đọc.

- [0. Bài toán & ý tưởng kiến trúc](AVAILABILITY-ALLOCATOR-OVERVIEW.md#0-bài-toán--ý-tưởng-kiến-trúc)
- [1. Một engine chung cho mọi kênh](AVAILABILITY-ALLOCATOR-OVERVIEW.md#1-một-engine-chung-cho-mọi-kênh)
- [2. Availability được tính như thế nào](AVAILABILITY-ALLOCATOR-OVERVIEW.md#2-availability-được-tính-như-thế-nào)
- [3. Chọn bàn tự động — tightest fit](AVAILABILITY-ALLOCATOR-OVERVIEW.md#3-chọn-bàn-tự-động--tightest-fit)
- [4. Flow hold → confirm (widget)](AVAILABILITY-ALLOCATOR-OVERVIEW.md#4-flow-hold--confirm-widget)
- [5. Chống trùng booking giữa các kênh (điểm mấu chốt)](AVAILABILITY-ALLOCATOR-OVERVIEW.md#5-chống-trùng-booking-giữa-các-kênh-điểm-mấu-chốt)
- [6. Hiệu năng tầng đọc](AVAILABILITY-ALLOCATOR-OVERVIEW.md#6-hiệu-năng-tầng-đọc)
- [7. Closeout — hai mode](AVAILABILITY-ALLOCATOR-OVERVIEW.md#7-closeout--hai-mode)

### 15. [Kiến trúc API: RESTful, GraphQL, gRPC](API_ARCHITECTURES.md)

Ba kiến trúc API phổ biến: thành phần cấu tạo, luồng giao tiếp, ví dụ tối giản, chuyện gì xảy ra khi gọi, và tiêu chí lựa chọn.

- [1. Tổng quan nhanh](API_ARCHITECTURES.md#1-tổng-quan-nhanh)
- [2. RESTful](API_ARCHITECTURES.md#2-restful)
  - Thành phần · Diagram giao tiếp · Ví dụ · Khi gọi, chuyện gì xảy ra · Ưu / nhược điểm · Dùng khi nào
- [3. GraphQL](API_ARCHITECTURES.md#3-graphql)
  - Thành phần · Diagram giao tiếp · Ví dụ · Khi gọi, chuyện gì xảy ra · Ưu / nhược điểm · Dùng khi nào
- [4. gRPC](API_ARCHITECTURES.md#4-grpc)
  - Thành phần · Diagram giao tiếp · Ví dụ · Khi gọi, chuyện gì xảy ra · Ưu / nhược điểm · Khi dùng
- [5. Chọn kiến trúc nào](API_ARCHITECTURES.md#5-chọn-kiến-trúc-nào)

### 16. [Apache Kafka — từ khái niệm đến kiến trúc](KAFKA-ARCHITECTURE.md)

Kafka là gì, sáu khái niệm cốt lõi, kiến trúc bên trong (replication, write/read path, storage, rebalance, delivery semantics) và khi nào nên/không nên dùng.

- [1. Kafka là gì?](KAFKA-ARCHITECTURE.md#1-kafka-là-gì)
- [2. Sáu khái niệm cốt lõi](KAFKA-ARCHITECTURE.md#2-sáu-khái-niệm-cốt-lõi)
  - [Partition và offset](KAFKA-ARCHITECTURE.md#partition-và-offset--nhìn-tận-mắt) · [Consumer group](KAFKA-ARCHITECTURE.md#consumer-group--chia-việc-và-nhân-bản-luồng-đọc)
- [3. Kiến trúc bên trong](KAFKA-ARCHITECTURE.md#3-kiến-trúc-bên-trong)
  - [Bức tranh tổng thể](KAFKA-ARCHITECTURE.md#31-bức-tranh-tổng-thể) · [Partition trên cluster](KAFKA-ARCHITECTURE.md#32-partition-trải-ra-trên-cluster-như-thế-nào) · [Replication](KAFKA-ARCHITECTURE.md#33-replication--leader-và-follower) · [Write/read path](KAFKA-ARCHITECTURE.md#34-đường-đi-của-một-bản-ghi-write-path--read-path) · [Segment & retention](KAFKA-ARCHITECTURE.md#35-lưu-trữ-trên-đĩa--segment-và-retention) · [Group coordinator & rebalance](KAFKA-ARCHITECTURE.md#36-group-coordinator-và-rebalance) · [Delivery semantics](KAFKA-ARCHITECTURE.md#37-ngữ-nghĩa-giao-nhận-delivery-semantics)
- [4. Kafka dùng để làm gì?](KAFKA-ARCHITECTURE.md#4-kafka-dùng-để-làm-gì)
- [5. Khi nào _không_ cần Kafka?](KAFKA-ARCHITECTURE.md#5-khi-nào-không-cần-kafka)
- [6. Tóm tắt](KAFKA-ARCHITECTURE.md#6-tóm-tắt)

### 17. [Các mô hình song song hóa trong Node.js — worker_threads, cluster, đa process](NODE_CONCURRENCY_MODELS.md)

Ba cách vượt giới hạn single-thread của Node.js: worker_threads cho tác vụ CPU-bound, cluster để chia tải request, và đa process độc lập theo vai trò; so sánh, tiêu chí chọn và đối chiếu với nollie-api.

- [1. Mô hình tư duy tối giản](NODE_CONCURRENCY_MODELS.md#1-mô-hình-tư-duy-tối-giản)
- [2. Worker Threads](NODE_CONCURRENCY_MODELS.md#2-worker-threads)
- [3. Cluster](NODE_CONCURRENCY_MODELS.md#3-cluster)
- [4. Đa process độc lập (multi-process theo vai trò)](NODE_CONCURRENCY_MODELS.md#4-đa-process-độc-lập-multi-process-theo-vai-trò)
- [5. So sánh trực tiếp](NODE_CONCURRENCY_MODELS.md#5-so-sánh-trực-tiếp)
- [6. Khi nào dùng cái nào](NODE_CONCURRENCY_MODELS.md#6-khi-nào-dùng-cái-nào)
- [7. Đối chiếu với nollie-api](NODE_CONCURRENCY_MODELS.md#7-đối-chiếu-với-nollie-api)
- [Nguồn tham khảo](NODE_CONCURRENCY_MODELS.md#nguồn-tham-khảo)

### 18. [Clean Architecture](CLEAN-ARCHITECTURE.md)

Mô hình 4 lớp của Uncle Bob (Entities → Use Cases → Adapters → Frameworks), Dependency Rule, ví dụ cụ thể theo từng file cho module booking, và đối chiếu với layered architecture của nollie-api.

- [1. Định nghĩa](CLEAN-ARCHITECTURE.md#1-định-nghĩa)
- [2. Các lớp (từ trong ra ngoài)](CLEAN-ARCHITECTURE.md#2-các-lớp-từ-trong-ra-ngoài)
- [3. Dependency Rule — quy tắc quan trọng nhất](CLEAN-ARCHITECTURE.md#3-dependency-rule--quy-tắc-quan-trọng-nhất)
- [4. Ví dụ cụ thể — feature "booking" theo Clean Architecture](CLEAN-ARCHITECTURE.md#4-ví-dụ-cụ-thể--feature-booking-theo-clean-architecture)
  - Cấu trúc thư mục · Luồng request qua từng file · Mã nguồn từng file · Điểm mấu chốt
- [5. Lợi ích và cái giá](CLEAN-ARCHITECTURE.md#5-lợi-ích-và-cái-giá)
- [6. So sánh với kiến trúc của nollie-api](CLEAN-ARCHITECTURE.md#6-so-sánh-với-kiến-trúc-của-nollie-api)
- [7. Khi nào nên áp dụng đầy đủ](CLEAN-ARCHITECTURE.md#7-khi-nào-nên-áp-dụng-đầy-đủ)
- [8. Tài liệu tham khảo](CLEAN-ARCHITECTURE.md#8-tài-liệu-tham-khảo)

### 19. [Bảo vệ Public API — Kỹ thuật & Áp dụng trong Booking Widget](PUBLIC-API-PROTECTION.md)

Các lớp phòng thủ cho endpoint không cần đăng nhập: rate limiting nhiều tầng, chống IDOR bằng slug resolution, uniform 404, capability token, idempotency, CAPTCHA, cache chắn DB — đối chiếu trực tiếp với module `booking-widget`.

- [1. Bản đồ các lớp phòng thủ](PUBLIC-API-PROTECTION.md#1-bản-đồ-các-lớp-phòng-thủ)
- [2. Các kỹ thuật đang áp dụng trong dự án](PUBLIC-API-PROTECTION.md#2-các-kỹ-thuật-đang-áp-dụng-trong-dự-án)
  - [Rate limiting nhiều tầng](PUBLIC-API-PROTECTION.md#21-rate-limiting-nhiều-tầng) · [Server tự resolve resource — chống IDOR](PUBLIC-API-PROTECTION.md#22-server-tự-resolve-resource--không-tin-định-danh-từ-client) · [Uniform 404](PUBLIC-API-PROTECTION.md#23-uniform-404--chống-enumeration) · [Capability token](PUBLIC-API-PROTECTION.md#24-capability-token--thay-thế-đăng-nhập-cho-khách-vãng-lai) · [Idempotency key](PUBLIC-API-PROTECTION.md#25-idempotency-key--an-toàn-trước-double-submit)
  - [Input validation](PUBLIC-API-PROTECTION.md#26-input-validation) · [CAPTCHA](PUBLIC-API-PROTECTION.md#27-captcha--chặn-bot-ở-các-điểm-nóng) · [Caching](PUBLIC-API-PROTECTION.md#28-caching--bảo-vệ-database-khỏi-bề-mặt-đọc-public) · [Concurrency control](PUBLIC-API-PROTECTION.md#29-concurrency-control--đúng-đắn-dưới-tải-ghi-đồng-thời) · [Error handling & logging](PUBLIC-API-PROTECTION.md#210-error-handling--logging-trên-bề-mặt-public)
- [3. Các lớp nằm ngoài tầng ứng dụng](PUBLIC-API-PROTECTION.md#3-các-lớp-nằm-ngoài-tầng-ứng-dụng)
- [4. Các kỹ thuật phổ biến khác (tham khảo)](PUBLIC-API-PROTECTION.md#4-các-kỹ-thuật-phổ-biến-khác-dự-án-chưa-dùng--tham-khảo) — phân biệt bề mặt ẩn danh vs Internet-facing có định danh
  - [Nhóm A — bề mặt ẩn danh](PUBLIC-API-PROTECTION.md#41-nhóm-a--dùng-được-cho-bề-mặt-ẩn-danh): [Bot detection](PUBLIC-API-PROTECTION.md#bot-detection--device-fingerprinting) · [Signed URL/JWT](PUBLIC-API-PROTECTION.md#signed-url--token-tự-chứa-có-thời-hạn) · [Query cost limiting](PUBLIC-API-PROTECTION.md#giới-hạn-chi-phí-truy-vấn-query-cost-limiting) · [Security headers](PUBLIC-API-PROTECTION.md#security-headers--https-only) · [Geo-blocking](PUBLIC-API-PROTECTION.md#geo-blocking)
  - [Nhóm B — caller có định danh](PUBLIC-API-PROTECTION.md#42-nhóm-b--chỉ-dành-cho-caller-có-định-danh-server-to-server): [API key + quota](PUBLIC-API-PROTECTION.md#api-key--quota-theo-tier) · [HMAC signing](PUBLIC-API-PROTECTION.md#hmac-request-signing) · [Chống replay](PUBLIC-API-PROTECTION.md#chống-replay-timestamp--nonce) · [mTLS](PUBLIC-API-PROTECTION.md#mtls-mutual-tls) · [IP allowlist](PUBLIC-API-PROTECTION.md#ip-allowlist)
- [5. Checklist khi thêm một endpoint public mới](PUBLIC-API-PROTECTION.md#5-checklist-khi-thêm-một-endpoint-public-mới)

### 21. [Pagination — Offset, Keyset/Cursor và thiết kế API phân trang](PAGINATION.md)

Lý thuyết chung về phân trang: vì sao offset chậm dần và trôi trang, vì sao keyset ổn định, khi nào cần opaque cursor; response envelope, các quy tắc an toàn và bảng chọn mô hình — nollie-api làm ví dụ cho cả ba mô hình.

- [1. Vì sao phải phân trang](PAGINATION.md#1-vì-sao-phải-phân-trang)
- [2. Nguyên tắc nền: thứ tự phải deterministic](PAGINATION.md#2-nguyên-tắc-nền-thứ-tự-phải-deterministic)
  - Không có ORDER BY · Thiếu tie-breaker trên cột không unique · Khóa sort mutable
- [3. Ba mô hình phân trang](PAGINATION.md#3-ba-mô-hình-phân-trang)
  - [Offset — "trang số N"](PAGINATION.md#31-offset-pagination--trang-số-n) · [Keyset/cursor — "tiếp sau bản ghi X"](PAGINATION.md#32-keyset-cursor-pagination--tiếp-sau-bản-ghi-x) · [Opaque cursor — keyset giấu sau token](PAGINATION.md#33-opaque-cursor--keyset-giấu-sau-một-token) · [Bảng so sánh ba mô hình](PAGINATION.md#34-bảng-so-sánh-ba-mô-hình) · [Keyset nâng cao — nhiều cột sort, filter và search](PAGINATION.md#35-keyset-nâng-cao--nhiều-cột-sort-filter-và-search)
- [4. Thiết kế response envelope](PAGINATION.md#4-thiết-kế-response-envelope)
- [5. Các quy tắc an toàn đi kèm](PAGINATION.md#5-các-quy-tắc-an-toàn-đi-kèm)
- [6. Chọn mô hình nào — bảng quyết định](PAGINATION.md#6-chọn-mô-hình-nào--bảng-quyết-định)
- [7. Checklist khi thêm một endpoint trả danh sách](PAGINATION.md#7-checklist-khi-thêm-một-endpoint-trả-danh-sách)

### 22. [GitHub Actions — Cheatsheet cú pháp chung](GITHUB-ACTIONS-CHEATSHEET.md)

Tổng hợp nhanh các cú pháp và pattern hay dùng trong GitHub Actions: trigger, permissions, jobs, steps, outputs, và các biến context phổ biến.

- [1. Cấu trúc khung tổng quát](GITHUB-ACTIONS-CHEATSHEET.md#1-cấu-trúc-khung-tổng-quát)
- [2. `on` — sự kiện trigger](GITHUB-ACTIONS-CHEATSHEET.md#2-on--sự-kiện-trigger)
- [3. `permissions` — quyền của `GITHUB_TOKEN`](GITHUB-ACTIONS-CHEATSHEET.md#3-permissions--quyền-của-github_token)
- [4. `jobs` — các job và quan hệ giữa chúng](GITHUB-ACTIONS-CHEATSHEET.md#4-jobs--các-job-và-quan-hệ-giữa-chúng)
- [5. `steps` — các bước trong một job](GITHUB-ACTIONS-CHEATSHEET.md#5-steps--các-bước-trong-1-job)
- [6. Biến & context hay dùng](GITHUB-ACTIONS-CHEATSHEET.md#6-biến--context-hay-dùng)
