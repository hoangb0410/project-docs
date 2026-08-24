# BẢO VỆ PUBLIC API: KỸ THUẬT & ÁP DỤNG TRONG BOOKING WIDGET

Public API là các endpoint không yêu cầu đăng nhập — bất kỳ ai trên Internet cũng gọi được. Trong dự án, bề mặt public lớn nhất là **booking widget** (`src/modules/booking-widget/`), phục vụ khách vãng lai đặt bàn qua trang web của nhà hàng. Tài liệu này tổng hợp các lớp phòng thủ: lý thuyết của từng kỹ thuật, và cách dự án đang áp dụng cụ thể.

Nguyên tắc xuyên suốt: **không có một lớp nào đủ một mình** — bảo vệ public API là phòng thủ theo chiều sâu (defense in depth). Mỗi lớp chặn một loại rủi ro khác nhau, và lớp sau giả định lớp trước đã bị vượt qua.

Cấu trúc tài liệu:

- **Mục 1** — bản đồ tổng quan các lớp phòng thủ.
- **Mục 2** — các kỹ thuật dự án đang áp dụng (lý thuyết + đối chiếu code).
- **Mục 3** — các lớp nằm ngoài tầng ứng dụng, ghi lại để không ngộ nhận.
- **Mục 4** — các kỹ thuật phổ biến khác dự án chưa dùng, để tham khảo.
- **Mục 5** — checklist khi thêm endpoint public mới.

---

## 1. Bản đồ các lớp phòng thủ

| Lớp | Rủi ro chặn được | Hiện thực trong dự án |
| --- | --- | --- |
| Rate limiting nhiều tầng | Spam, brute-force, DoS mức ứng dụng | `ThrottlerModule` + Redis storage, 3 tier |
| Resource resolution phía server | IDOR (đọc/ghi dữ liệu venue khác) | `PublicWidgetGuard` — slug → venue, cấm client gửi `venueId` |
| Uniform 404 | Dò quét (enumeration) venue tồn tại | Mọi trường hợp fail đều trả 404 giống nhau |
| Capability token | Truy cập booking không cần tài khoản | Token UUID opaque cho hold / manage / waitlist offer |
| Idempotency key | Double-submit, retry tạo bản ghi trùng | Header `idempotency-key` trên POST holds |
| Input validation | Payload rác, injection, mass assignment | DTO + `I18nValidationPipe` toàn cục |
| CAPTCHA | Bot tự động hóa | `CaptchaGuard` (Cloudflare Turnstile) — hiện dùng cho auth |
| Caching | Đánh sập DB qua endpoint đọc | Redis cache slug + widget read cache |
| Concurrency control | Race condition khi ghi (double booking) | Pacing lock, hold có `expiresAt` |
| Error handling thống nhất | Lộ thông tin nội bộ qua lỗi | `HttpExceptionFilter` toàn cục |

---

## 2. Các kỹ thuật đang áp dụng trong dự án

Widget là **bề mặt ẩn danh** (theo phân loại ở mục 4): khách không đăng nhập và không giữ secret nào trước khi gọi. Vì vậy toàn bộ kỹ thuật trong mục này đều hoạt động mà không cần caller có định danh — phòng thủ dựa vào kiểm soát hành vi và tài nguyên, không dựa vào danh tính. Hai điểm cần đọc đúng:

- **Capability token (2.4) không phải ngoại lệ.** Token trông giống "định danh" nhưng do server phát ra *sau* một hành động ẩn danh (đặt bàn xong mới có), không phải credential đăng ký trước; nó là ủy quyền hẹp trên đúng một tài nguyên, không phải danh tính của caller.
- **Tier `agentKey` (bảng ở 2.1) là ngoại lệ duy nhất.** Nó xuất hiện vì cấu hình throttler dùng chung một file cho mọi app, nhưng bản chất phục vụ agent API — bề mặt có định danh (nhóm B, mục 4.2), không phải widget.

### 2.1. Rate limiting nhiều tầng

**Lý thuyết.** Rate limiting giới hạn số request trong một cửa sổ thời gian. Điểm quan trọng là **chọn đúng khóa đếm (tracker)**:

- **Theo IP** — chặn một client spam, nhưng vô dụng trước botnet phân tán và có thể chặn oan nhiều người sau cùng một NAT.
- **Theo resource (venue, tài khoản)** — chặn tổng tải dồn vào một tài nguyên, bất kể đến từ bao nhiêu IP.
- **Theo credential (API key)** — gắn hạn mức với danh tính đã cấp phát.

Một hệ thống tốt dùng **nhiều tầng đồng thời**: request phải qua được tất cả các tầng áp dụng cho nó. Bộ đếm phải nằm ở **storage chia sẻ (Redis)**, không nằm trong RAM của từng instance — nếu không, chạy N instance sau load balancer nghĩa là hạn mức thực tế bị nhân N lần.

Khi từ chối, trả `429` kèm header `x-ratelimit-*` để client hợp lệ biết lùi lại (backoff) thay vì retry mù.

**Áp dụng trong dự án.** Ba tier khai báo tại `src/configs/throttler-option.ts`, dùng chung cho app chính và widget app standalone:

| Tier | Tracker | Mặc định | Ghi chú |
| --- | --- | --- | --- |
| `default` | IP | 6 req/phút | Từng endpoint override bằng `@Throttle` |
| `venue` | `venue:{venueId}` | 600 req/phút | Chỉ chạy khi request đã có `resolvedVenue` |
| `agentKey` | `agent-key:{id}` | 120 req/phút | Cho agent API, theo API key |

- Storage là Redis (`ThrottlerRedisStorage`) — bộ đếm đúng khi scale ngang.
- Từng endpoint đặt hạn mức riêng theo độ nhạy: đọc availability rộng hơn, còn `checkout-session` và `invite` chỉ 5 req/phút.
- `WidgetThrottlerGuard` (`src/utils/guards/widget-throttler.guard.ts`) kế thừa `ThrottlerGuard` để set header `x-ratelimit-limit/remaining/reset` khi trả 429; tier venue có hậu tố `-venue` để client phân biệt bị chặn theo IP hay theo venue.

**Bài học thiết kế**: tier venue tồn tại vì tier IP không bảo vệ được venue trước traffic phân tán — hai tier trả lời hai câu hỏi khác nhau ("client này có spam không?" và "venue này có đang bị dồn tải không?").

### 2.2. Server tự resolve resource — không tin định danh từ client

**Lý thuyết.** Lỗ hổng phổ biến nhất của public API là **IDOR (Insecure Direct Object Reference)**: client gửi `venueId=123`, server dùng luôn giá trị đó để truy vấn. Kẻ tấn công chỉ cần đổi số là đọc/ghi dữ liệu của venue khác.

Cách chặn triệt để: **client chỉ được gửi định danh public (slug), server tự resolve ra ID nội bộ** và gắn vào request context. Mọi tầng phía sau chỉ đọc từ context đó, không bao giờ đọc ID từ body/query/header.

Mức chặt hơn: **chủ động reject** nếu client cố gửi ID nội bộ — biến mọi nỗ lực dò IDOR thành lỗi ngay tại cửa, đồng thời ngăn dev sau này vô tình đọc nhầm `body.venueId`.

**Áp dụng trong dự án.** `PublicWidgetGuard` (`src/utils/guards/public-widget.guard.ts`) chạy trước mọi handler widget (gắn qua decorator `@PublicWidget()`):

1. Lấy `slug` từ URL, resolve sang venue qua `venues.slug` (cache Redis 1 giờ, key `widget:venue:slug:{slug}`).
2. `assertNoClientVenueId` — trả 400 nếu client gửi `venueId` trong body, query, hoặc header `x-venue-id`.
3. Chỉ cho qua venue có `venueBookingMode = NOLLIE_BOOKINGS`; mode khác trả 404.
4. Gắn `request.resolvedVenue = { venueId, country, timezone, slug }` — service layer chỉ nhận venue từ đây.

Kết quả: toàn bộ truy vấn phía sau đều scope theo `venueId` do server resolve, đúng quy tắc "always scope to venueId" trong `CLAUDE.md`, và client không có đường nào tiêm ID.

### 2.3. Uniform 404 — chống enumeration

**Lý thuyết.** Nếu API trả lỗi khác nhau cho "không tồn tại" và "tồn tại nhưng không đủ điều kiện", kẻ tấn công dò được danh sách resource hợp lệ (enumeration). Nguyên tắc: **mọi lý do từ chối trên bề mặt public đều trả cùng một lỗi**, không phân biệt được từ bên ngoài.

**Áp dụng trong dự án.** Trong `PublicWidgetGuard`, cả ba trường hợp — slug thiếu, venue không tồn tại, venue tồn tại nhưng sai booking mode — đều trả 404 với cùng message `BOOKING_WIDGET.NOT_FOUND`. Từ ngoài nhìn vào, không thể phân biệt "nhà hàng này không dùng Nollie" với "slug này không tồn tại".

Lỗi đi qua i18n (`localesService.translate`) và `HttpExceptionFilter` toàn cục, nên response không bao giờ chứa stack trace hay chi tiết nội bộ.

### 2.4. Capability token — thay thế đăng nhập cho khách vãng lai

**Lý thuyết.** Khách đặt bàn không có tài khoản, nhưng vẫn cần xem/sửa/hủy booking của mình. Giải pháp là **capability token**: một token opaque, đủ dài để không đoán được (UUID v4 ≈ 122 bit ngẫu nhiên), và **bản thân việc sở hữu token chính là quyền truy cập** — đúng một booking, không hơn.

Yêu cầu với capability token:

- **Không đoán được** — sinh từ nguồn ngẫu nhiên mật mã, không phải ID tuần tự hay hash của email.
- **Phạm vi hẹp** — token của booking A không mở được booking B.
- **Có vòng đời** — token gắn với trạng thái nghiệp vụ (hold hết hạn thì token chết theo).
- **Đi kèm rate limit** — chặn brute-force dò token (dù xác suất trúng UUID gần bằng 0, vẫn phải chặn để không tốn tài nguyên).

**Áp dụng trong dự án.** Widget dùng ba loại token, đều là UUID opaque trên URL:

| Token | Endpoint | Quyền |
| --- | --- | --- |
| `holdToken` | `POST /holds`, chuyển tiếp vào `POST /bookings` | Giữ slot trong lúc điền form; hold có `expiresAt`, hết hạn tự vô hiệu |
| Manage token | `GET/PATCH /bookings/:token`, `/confirm`, `/calendar.ics`, `/feedback`, `/checkout-session`, `/invite` | Tự phục vụ trên đúng một booking |
| `offerToken` | `/waitlist/offers/:offerToken/accept\|decline` | Trả lời đúng một lời mời waitlist |

- Controller validate định dạng ngay tại cửa bằng `ParseUUIDPipe` — token sai format bị chặn trước khi chạm service.
- Mọi endpoint theo token đều có `WidgetThrottlerGuard` + `@Throttle` riêng (đọc 30/phút, ghi 5–10/phút) — brute-force dò token bị bóp từ tầng rate limit.
- Hold luôn được kiểm tra `expiresAt > now` khi sử dụng (`reservation-hold.service.ts`) — token quá hạn không còn giá trị.

### 2.5. Idempotency key — an toàn trước double-submit

**Lý thuyết.** Public API phía trước là mạng di động chập chờn: client sẽ retry. Với endpoint ghi (tạo hold, tạo booking), retry mà không có cơ chế chống trùng nghĩa là hai bản ghi. **Idempotency key** do client sinh; server ghi nhớ key đã xử lý và trả lại kết quả cũ thay vì tạo mới.

**Áp dụng trong dự án.** `POST /venues/:slug/holds` nhận header `idempotency-key` (bắt buộc — service reject nếu thiếu, `booking-widget.service.ts`). Retry cùng key trả về hold đã tạo, không giữ thêm slot. Phía agent API dùng cùng cơ chế qua idempotency interceptor với scope theo key.

### 2.6. Input validation

**Lý thuyết.** Mọi input từ public API là hostile cho đến khi chứng minh ngược lại. Validate bằng **schema khai báo (DTO)** ngay tại biên, trước khi dữ liệu chạm business logic:

- Kiểu, độ dài, format, enum — chặn payload rác và các vector injection.
- Chặn field lạ (mass assignment) — client không được gửi field mà DTO không khai báo, tránh trường hợp ghi đè cột nhạy cảm qua spread object vào model.
- Với SQL, luôn dùng parameterized query — không bao giờ nối chuỗi.

**Áp dụng trong dự án.**

- `I18nValidationPipe` đăng ký toàn cục trong `src/main.ts` — mọi DTO của widget (`src/modules/booking-widget/dto/`) được validate bằng class-validator trước khi vào controller.
- Sequelize + repository pattern (`BaseRepository`) — không có raw SQL nối chuỗi trên đường đi của widget.
- Param path validate bằng pipe (`ParseUUIDPipe` cho token).

### 2.7. CAPTCHA — chặn bot ở các điểm nóng

**Lý thuyết.** Rate limit chặn *tần suất*, không chặn *tự động hóa*: bot chạy dưới ngưỡng vẫn lọt. CAPTCHA (dạng hiện đại như Cloudflare Turnstile chấm điểm ngầm, không bắt người dùng giải đố) thêm chi phí đáng kể cho automation ở các điểm nóng: đăng ký, đăng nhập, form public tạo dữ liệu.

Nguyên tắc verify phía server: token CAPTCHA phải được server xác thực với nhà cung cấp (kèm IP client), không bao giờ tin kết quả do frontend tự khai.

**Áp dụng trong dự án.** `CaptchaGuard` (`src/utils/guards/captcha.guard.ts`) verify token Cloudflare Turnstile qua endpoint `siteverify`, kèm `remoteip`. Hiện gắn trên các endpoint auth (`auth.controller.ts`) — chưa nằm trên widget. Nếu widget xuất hiện abuse dạng bot tạo booking/waitlist rác vượt qua rate limit, guard này là lớp có sẵn để gắn thêm vào các POST nhạy cảm mà không phải xây mới.

### 2.8. Caching — bảo vệ database khỏi bề mặt đọc public

**Lý thuyết.** Endpoint đọc public (config, availability) là mục tiêu dễ nhất để đánh sập DB: hợp lệ từng request, nhưng dồn số lượng. Cache có TTL đứng chắn giữa Internet và DB: kẻ tấn công chỉ hâm nóng cache thay vì đốt DB. Cache ở đây là **biện pháp bảo vệ hạ tầng**, không chỉ là tối ưu tốc độ.

Điểm cần cẩn thận: **cache key phải bao trọn mọi tham số ảnh hưởng kết quả**, và phải có cơ chế invalidation khi dữ liệu nguồn đổi (dự án dùng venue-version key để vô hiệu hàng loạt — xem `CACHING-PATTERNS.md`).

**Áp dụng trong dự án.**

- Slug → venue cache 1 giờ trong `PublicWidgetGuard` — truy vấn `venues` không bị lặp lại theo từng request.
- `widget-frontend-cache` (`src/services/widget-frontend-cache/`) cache các response đọc của widget; invalidation theo venue-version (`invalidateForVenue`).
- Đường đọc widget có SLA < 2 giây cho availability — cache là một phần của cam kết đó.

### 2.9. Concurrency control — đúng đắn dưới tải ghi đồng thời

**Lý thuyết.** Public API ghi mà nhiều client cùng nhắm một tài nguyên (slot đặt bàn) sẽ gặp race condition: hai khách cùng "thấy còn chỗ" và cùng đặt. Chống bằng lock có phạm vi hẹp (đúng tài nguyên tranh chấp) + trạng thái trung gian có thời hạn (hold), thay vì lock rộng làm nghẽn toàn hệ thống. Chi tiết lý thuyết ở `RACE-CONDITION-HANDBOOK.md` và `CONCURRENCY-CONTROL.md`.

**Áp dụng trong dự án.**

- Pacing lock theo `(venue, service, date)` dùng chung giữa widget confirm và agent create — hai đường vào không thể cùng vượt pacing.
- Hold giữ slot có `expiresAt`; khách bỏ ngang thì slot tự giải phóng, không cần dọn tay.
- Hệ quả chấp nhận được (đã quyết định): `/availability` dựa trên pacing, `/hold` mới kiểm tra bàn vật lý — hold có thể trả `410 SLOT_UNAVAILABLE` trên slot "available".

### 2.10. Error handling & logging trên bề mặt public

**Lý thuyết.** Lỗi trên public API là kênh rò rỉ thông tin: stack trace lộ framework, message lộ cấu trúc DB, timing lộ tồn tại của dữ liệu. Nguyên tắc:

- Mọi lỗi đi qua một filter toàn cục, trả shape thống nhất, không bao giờ kèm stack trace.
- Log đầy đủ **ở phía server** để debug, nhưng response chỉ chứa code + message an toàn.
- Không log PII của khách trên đường public (bắt buộc với khách UK theo quy tắc dự án — chỉ log ID).

**Áp dụng trong dự án.**

- `HttpExceptionFilter` (`src/utils/filters/http-exception.filter.ts`) là chốt chặn cuối: log lỗi và trả response chuẩn hóa. Vì vậy các method đọc của widget **cố ý không có** try-catch log riêng từng call — đây là quyết định đã review, không phải thiếu sót.
- Message lỗi đi qua i18n (`localesService.translate` + key trong `src/constants/messages`), không hardcode chuỗi lộ chi tiết nội bộ.

---

## 3. Các lớp nằm ngoài tầng ứng dụng

Ghi lại để không ngộ nhận là "đã được bảo vệ":

- **CORS hiện mở hoàn toàn** — `app.enableCors()` trong `src/main.ts` không giới hạn origin. Với widget nhúng trên domain của từng nhà hàng, CORS mở là chủ đích (không biết trước origin); nhưng cần hiểu đúng: **CORS không phải cơ chế bảo mật server** — nó chỉ ràng buộc browser, không chặn được curl/bot. Các lớp ở mục 2 mới là phòng thủ thật.
- **Chưa có helmet / security headers** ở tầng NestJS — nếu cần (chủ yếu cho response HTML, ít tác dụng với JSON API), thêm ở tầng edge hoặc middleware. Chi tiết ở mục 4.1.
- **WAF / chống DDoS mức mạng** thuộc về tầng hạ tầng (Cloudflare / ALB), không nằm trong codebase. Rate limit ứng dụng chặn abuse mức nghiệp vụ, không chặn được volumetric DDoS.

---

## 4. Các kỹ thuật phổ biến khác (dự án chưa dùng — tham khảo)

Mục 2 bám theo những gì booking widget thực dùng. Trước khi liệt kê phần còn lại, cần tách bạch một điểm hay bị trộn lẫn: **"public API" có hai nghĩa khác nhau**, và mỗi nghĩa đi với một bộ kỹ thuật khác nhau.

| | Bề mặt ẩn danh | Bề mặt Internet-facing có định danh |
| --- | --- | --- |
| Ai gọi được | Bất kỳ ai, không cần đăng ký | Bất kỳ ai *chạm* được, nhưng chỉ caller đã đăng ký *gọi* được |
| Caller giữ được secret không | **Không** — code chạy trong browser, mọi thứ trong JS là công khai | **Có** — secret nằm ở server của caller |
| Ví dụ | Booking widget | Stripe API; trong dự án là agent API (`x-api-key`) |
| Phòng thủ dựa vào | Kiểm soát hành vi + resource (rate limit, CAPTCHA, capability token) | Kiểm soát danh tính (key, chữ ký, cert) |

Hệ quả: FE **không thể** "tự sinh" API key hay tự ký HMAC — secret nằm trong JS là công khai, ai đọc source cũng lấy được. Nhóm kỹ thuật dựa trên secret (4.2) vì vậy chỉ dành cho nghĩa thứ hai, không gắn được vào widget.

### 4.1. Nhóm A — dùng được cho bề mặt ẩn danh

#### Bot detection & device fingerprinting

- **Chặn gì:** bot có tổ chức mà rate limit theo IP không đỡ nổi (botnet đổi IP liên tục, scraping chậm rãi dưới ngưỡng).
- **Cách hoạt động:** edge thu tín hiệu trên từng request — TLS fingerprint (JA3), thứ tự header, tốc độ thao tác, danh tiếng IP — rồi chấm một điểm bot. Theo ngưỡng điểm mà phản ứng mềm dần: cho qua → làm chậm (tarpit) → bắt challenge → chặn. Khác CAPTCHA (chặn tại một điểm), đây là chấm điểm liên tục.
- **Cài đặt:** không tự xây — bật dịch vụ ở tầng edge (Cloudflare Bot Management, DataDome). Backend nếu muốn phản ứng riêng thì đọc score do edge gắn vào header (ví dụ `cf-bot-score`) và tự quyết định chặn/ghi log.
- **Dùng khi:** bề mặt bị abuse có tổ chức; đừng mua sớm khi rate limit còn đỡ được.

#### Signed URL / token tự chứa có thời hạn

- **Chặn gì:** truy cập ngoài hạn/ngoài phạm vi vào resource read-only, mà không tốn một lượt tra DB mỗi request.
- **Cách hoạt động:** server ký payload `{resource, quyền, hết hạn}` bằng secret của server rồi phát cho client; client chỉ cầm và trình lại (vẫn là mô hình ẩn danh — **không phải client tự ký**). Khi nhận, server chỉ cần verify chữ ký + hạn, không chạm DB.
- **Cài đặt:** JWT hạn ngắn (`jsonwebtoken`: `sign({resource, scope}, secret, {expiresIn: '15m'})` / `verify`), hoặc S3 presigned URL cho file. Đánh đổi phải chấp nhận: **không thu hồi được trước hạn** (không có bản ghi nào để xóa) → hạn phải ngắn. Token opaque của widget (mục 2.4) là lựa chọn ngược lại: tra DB mỗi lần nhưng thu hồi được ngay.
- **Dùng khi:** read-only, hạn ngắn, verify tần suất cao (link tải file, ảnh CDN).

#### Giới hạn chi phí truy vấn (query cost limiting)

- **Chặn gì:** một request hợp lệ về *tần suất* nhưng là quả bom về *chi phí* — rate limit đếm số request, không đo độ đắt.
- **Cách hoạt động:** chặn mọi ngả client điều khiển được kích thước kết quả: số bản ghi (phân trang), độ sâu quan hệ (include/join), thời gian chạy (timeout).
- **Cài đặt:** cap `limit` ngay trong DTO (`@Max(1000)`, default 100 — dự án đã làm); đặt `statement_timeout` phía Postgres cho connection public; với GraphQL thêm depth limit + complexity scoring (`graphql-depth-limit`) — một query lồng 10 cấp có thể đắt bằng nghìn request REST.
- **Dùng khi:** luôn luôn, ở mức tối thiểu là cap phân trang.

#### Security headers & HTTPS-only

- **Chặn gì:** downgrade xuống HTTP, MIME sniffing, và các tấn công browser-side khi API serve nội dung render trực tiếp (trang quản lý booking, file `.ics`).
- **Cách hoạt động:** server gắn header chỉ thị cho browser — `Strict-Transport-Security` (từ nay chỉ HTTPS), `X-Content-Type-Options: nosniff`, `Content-Security-Policy` (nguồn script/style được phép).
- **Cài đặt:** `app.use(helmet())` trong `main.ts`, hoặc gắn ở edge/load balancer. Với API trả JSON thuần, giá trị thực chủ yếu là HSTS — đừng ảo tưởng các header còn lại bảo vệ được JSON API.
- **Dùng khi:** luôn luôn — chi phí gần bằng không.

#### Geo-blocking

- **Chặn gì:** noise và tấn công từ vùng địa lý mà sản phẩm không phục vụ.
- **Cách hoạt động:** edge tra IP → quốc gia, chặn trước khi request chạm server. Là lớp *giảm tải*, không phải bảo mật thật — VPN vượt được.
- **Cài đặt:** WAF rule ở Cloudflare, ví dụ `(not ip.geoip.country in {"GB" "AU" "NZ"}) → Block`; không viết trong code ứng dụng.
- **Dùng khi:** thị trường giới hạn địa lý rõ và đang chịu abuse từ ngoài vùng.

### 4.2. Nhóm B — chỉ dành cho caller có định danh (server-to-server)

Điều kiện chung: caller giữ được secret/cert ở **server của họ** — browser widget không làm được. Trong dự án, ứng viên tự nhiên là agent API và webhook.

#### API key + quota theo tier

- **Chặn gì:** consumer vô danh, và consumer hợp lệ nhưng dùng vượt gói.
- **Cách hoạt động:** cấp key khi consumer đăng ký; mỗi request trình key qua header, guard tra key → gắn context (consumer nào, gói nào). Quota là **hạn mức dài hạn** (10.000 call/tháng) đếm dồn theo kỳ, khác rate limit **chống burst ngắn hạn** (100 call/phút) — API thương mại cần cả hai.
- **Cài đặt:** lưu **hash** của key trong DB (không lưu plain — giống password); counter usage theo `(keyId, tháng)` trong Redis/DB, reject khi vượt. Dự án đã có nửa đầu: `agent_api_keys` + tier `agentKey` chống burst; quota dài hạn chưa có.
- **Dùng khi:** API mở cho đối tác đăng ký, phân gói dịch vụ, tính tiền theo usage.

#### HMAC request signing

- **Chặn gì:** giả mạo request (không có secret thì không ký được) và sửa payload trên đường đi — hai thứ API key trần không chặn.
- **Cách hoạt động:** caller và server chia sẻ secret. Caller ký từng request, server tính lại cùng công thức và so sánh:

```typescript
// Caller (server của đối tác) — và server verify tính lại đúng chuỗi này
const base = `${method}\n${path}\n${timestamp}\n${sha256(body)}`;
const signature = crypto.createHmac('sha256', secret).update(base).digest('hex');
// gửi kèm: x-signature, x-timestamp
```

- **Cài đặt:** so sánh bằng `crypto.timingSafeEqual` (chống timing attack, không dùng `===`); body phải lấy **raw bytes** trước khi JSON parse (parse rồi serialize lại có thể lệch từng byte). Đây chính là mô hình webhook signature của Stripe/SendGrid — dự án đang verify chiều nhận, chưa yêu cầu chiều caller ký.
- **Dùng khi:** server-to-server, payload có giá trị tài chính.

#### Chống replay: timestamp + nonce

- **Chặn gì:** kẻ bắt được một request hợp lệ (đã ký đúng) đem phát lại nguyên văn.
- **Cách hoạt động:** timestamp nằm trong chuỗi ký nên không sửa được → request quá cũ bị reject; nonce dùng một lần → request lặp bị reject:

```typescript
if (Math.abs(Date.now() - timestamp) > 300_000) reject(); // quá 5 phút
const fresh = await redis.set(`nonce:${nonce}`, 1, 'NX', 'EX', 300);
if (!fresh) reject(); // nonce đã dùng → replay
```

- **Cài đặt:** TTL của nonce chỉ cần bằng cửa sổ timestamp (request cũ hơn đã bị chặn ở điều kiện đầu, không cần nhớ nonce vĩnh viễn). Phân biệt với idempotency key (mục 2.5): idempotency chống retry *hiền* của client (trả lại kết quả cũ); nonce chống replay *ác* (reject thẳng).
- **Dùng khi:** đã có request signing và side-effect không đảo ngược được (chuyển tiền, cấp quyền).

#### mTLS (mutual TLS)

- **Chặn gì:** mọi kết nối không trình được certificate hợp lệ — bị loại ngay tầng TLS handshake, trước khi request chạm ứng dụng.
- **Cách hoạt động:** TLS hai chiều — client cũng phải trình cert do CA của mình phát hành; server verify chuỗi tin cậy khi bắt tay.
- **Cài đặt:** cấu hình ở load balancer/nginx (`ssl_verify_client on` + `ssl_client_certificate ca.pem`), ứng dụng chỉ đọc thông tin cert từ header do proxy gắn vào. Chi phí thật nằm ở vận hành: phát hành, xoay vòng, thu hồi cert cho từng client.
- **Dùng khi:** số client ít và kiểm soát được (nội bộ giữa service, đối tác ngân hàng/y tế).

#### IP allowlist

- **Chặn gì:** mọi nguồn ngoài dải IP đã biết.
- **Cách hoạt động:** so IP nguồn với danh sách CIDR cho phép, chặn trước khi vào ứng dụng.
- **Cài đặt:** đặt ở edge — security group / Cloudflare WAF — thay vì trong code; nếu buộc phải check trong app (sau proxy) thì lấy IP từ `x-forwarded-for` do **proxy tin cậy** gắn, không tin header client tự gửi.
- **Dùng khi:** webhook đến từ dải IP nhà cung cấp công bố, admin API chỉ gọi từ VPN công ty.

---

## 5. Checklist khi thêm một endpoint public mới

1. Gắn `@PublicWidget()` (hoặc `WidgetThrottlerGuard` cho route theo token) — không bao giờ để route public trần.
2. Đặt `@Throttle` theo độ nhạy: đọc rộng hơn ghi; endpoint tạo tiền/side-effect (checkout, invite) chặt nhất.
3. Không nhận ID nội bộ từ client — resolve từ slug/token, đọc venue từ `request.resolvedVenue`.
4. Mọi lý do từ chối trả cùng một 404 — không tạo nhánh lỗi phân biệt được từ ngoài.
5. Token mới phải là UUID sinh ngẫu nhiên, validate bằng `ParseUUIDPipe`, có vòng đời (hết hạn/one-shot).
6. Endpoint ghi có retry-risk → yêu cầu idempotency key.
7. DTO khai báo đầy đủ, không field thừa lọt qua.
8. Endpoint đọc gọi DB → cân nhắc cache + invalidation theo venue-version.
9. Không log PII khách UK; lỗi để `HttpExceptionFilter` xử lý, không tự trả message lộ nội bộ.
