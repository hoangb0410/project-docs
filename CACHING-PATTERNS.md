# Caching Patterns — Lý thuyết & Áp dụng

Tài liệu học về các pattern caching phía backend: khi nào cache, cache thế nào, và quan trọng nhất — **vô hiệu hóa cache thế nào cho đúng**. Lý thuyết viết chung để áp dụng được ở mọi dự án; ví dụ minh họa lấy từ widget đặt bàn của nollie-api (Redis + NestJS + Next.js).

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

---

## 1. Nền tảng: Cache-Aside (Lazy Loading)

### Lý thuyết

Pattern phổ biến nhất. Ứng dụng tự quản lý cache, DB không biết gì về cache:

```
1. Đọc cache theo key
2. HIT  → trả về luôn
3. MISS → query nguồn thật (DB) → ghi vào cache kèm TTL → trả về
```

Đặc điểm:
- **Lazy**: chỉ dữ liệu thực sự được đọc mới nằm trong cache — không tốn công warm dữ liệu không ai cần.
- **Chịu được cache chết**: Redis sập thì mọi request thành miss, hệ thống chậm đi nhưng vẫn đúng. Cache phải luôn là tầng *tăng tốc*, không bao giờ là tầng *sự thật*.
- **Nhược điểm cố hữu**: lần đọc đầu sau mỗi lần miss luôn chậm (phải tính lại), và có khoảng cửa sổ dữ liệu cũ nếu chỉ dựa vào TTL.

### Ví dụ trong dự án

`booking-widget.service.ts` → `getConfig()`: đọc `widget:config:{slug}` từ Redis, miss thì gom branding + services + areas + custom fields từ DB, set lại với TTL 1 giờ. Đúng nguyên bản cache-aside, không thêm gì.

```typescript
const cached = await this.readJsonCache(cacheKey);
if (cached) return cached;
// ... query DB, build config ...
await this.redisService.setWithTTL(cacheKey, JSON.stringify(config), 3600);
return config;
```

---

## 2. Chọn TTL: câu hỏi đầu tiên của mọi cache

### Lý thuyết

TTL không phải con số tùy hứng — nó trả lời câu hỏi: **"dữ liệu này được phép cũ tối đa bao lâu?"** Phân loại dữ liệu trước khi chọn:

| Loại dữ liệu | Tần suất đổi | Hậu quả khi stale | TTL phù hợp |
|---|---|---|---|
| Config-shaped (branding, danh sách dịch vụ, settings) | Hiếm — chỉ khi admin lưu | Thấp (hiển thị sai vài phút) | Dài (giờ) + invalidate chủ động khi ghi |
| State-shaped (tồn kho, availability, số dư) | Liên tục — mỗi transaction | Cao (bán trùng, overbook) | Ngắn (giây) + invalidate chủ động **bắt buộc** |

Nguyên tắc: **TTL chỉ là lưới an toàn (backstop), không phải cơ chế invalidation chính** cho dữ liệu state-shaped. Nếu bạn thấy mình chọn TTL = 5s để "cho đỡ stale", đó là dấu hiệu bạn đang thiếu invalidation chủ động.

### Ví dụ trong dự án

Widget có đúng hai loại này:
- `/config` (branding, services): TTL **1 giờ**, vì mọi chỗ admin ghi đều gọi refresh chủ động (mục 5) — TTL dài chỉ đỡ trường hợp ghi ngoài luồng.
- Availability/pacing: TTL **60 giây**, vì một booking mới phải làm slot biến mất *ngay*, không đợi nổi 60s — nên mọi write path đều invalidate chủ động (mục 3), TTL 60s chỉ là backstop khi có đường ghi nào đó bị bỏ sót.

---

## 3. Version-Keyed Invalidation (Generation-based)

### Vấn đề lý thuyết

Một thực thể logic thường phình ra thành **rất nhiều cache key** — mỗi tổ hợp tham số một key (userId × page × filter × ...). Khi thực thể đổi, làm sao xóa hết?

- **Cách ngây thơ**: `SCAN` toàn keyspace tìm pattern rồi `DEL` từng key. O(N) theo tổng số key trong Redis, block Redis, và Redis thường là shared resource — một lần invalidate làm chậm cả hệ thống.
- **Cách đúng**: đừng xóa. **Đổi tên nơi mọi người tìm đến.**

### Pattern

Toàn bộ pattern chỉ có **2 loại key**: một key **version** (con trỏ — chứa một con số đếm, tên cố định) và nhiều key **data** (đích — chứa dữ liệu thật, tên có nhúng con số đó). Reader đọc con trỏ để ghép ra tên đích; writer vô hiệu mọi đích bằng cách dịch con trỏ (+1) — các đích cũ lập tức mồ côi vì không ai ghép ra tên chúng nữa.

Cụ thể:

1. Nhúng một **số version** vào tên mọi cache key: `data:{params}:v{version}`.
2. Version lưu ở một key đếm riêng: `version:{entityId}`.
3. Reader luôn `GET version` trước, ghép vào key rồi mới đọc cache.
4. Invalidate = **`INCR` key version** ("bump"). Một lệnh, O(1), atomic, bất kể thực thể có bao nhiêu key con.

Hệ quả: key cũ (`...:v5`) vẫn nằm trong Redis nhưng **không reader nào trỏ tới nữa** — thành rác và tự bay theo TTL của chính nó. Đây là trade-off cốt lõi của pattern:

> **Đổi bộ nhớ lấy tốc độ invalidation.** Chấp nhận rác tồn tại ≤ TTL để invalidation là O(1). Vì vậy pattern này *đòi hỏi* data key có TTL ngắn — TTL dài thì rác chất đống.

Lưu ý thiết kế:
- Key version cũng cần TTL (dài hơn data TTL nhiều lần). Version hết hạn → đọc ra 0 → an toàn, vì data key nó từng trỏ tới đã hết hạn từ lâu.
- Version vắng mặt = 0, nghĩa là thực thể mới toanh chạy được ngay, không cần khởi tạo.

### Mở rộng: version nhiều tầng (composed version)

Khi invalidation có nhiều **phạm vi nổ (blast radius)** khác nhau — ví dụ "một ngày của một venue" vs "mọi ngày của venue" — đừng bắt caller bump N counter. Dùng nhiều counter và **ghép** thành một số:

```
version = coarseVersion × STRIDE + fineVersion
```

`STRIDE` đủ lớn để fine không bao giờ tràn sang coarse trong vòng đời TTL. Bump counter thô một lần → version ghép của *mọi* key con đều nhảy → tất cả vô hiệu cùng lúc, vẫn O(1).

### Ví dụ trong dự án

`allocator-cache.service.ts`:

```
# Data keys — mỗi tổ hợp partySize × serviceId một key
reservation:availability:{venueId}:{date}:{partySize}:{serviceId}:v{version}
pacing:{venueId}:{date}:svc:{serviceId}:v{version}

# Version counters
avail:ver:{venueId}:{date}    — bump khi có booking/hold ghi vào ngày đó
avail:venuever:{venueId}      — bump khi admin sửa service/area (ảnh hưởng mọi ngày)
```

```typescript
// Ghép 2 tầng: sửa config venue → mọi ngày vô hiệu bằng 1 INCR
version = venueVersion * 1_000_000 + dateVersion;
```

Data TTL 60s, version TTL 48h. Comment trong code ghi rõ pattern này **thay thế vòng SCAN+DEL cũ** — chính là bài học "cách ngây thơ" ở trên, dự án đã trả giá bằng một lần refactor.

---

## 4. Cache Stampede & Single-Flight Lock

### Vấn đề lý thuyết

**Stampede (thundering herd)**: khoảnh khắc một key hot bị vô hiệu (version bump hoặc TTL hết), *mọi* request đồng thời cùng miss và cùng lao vào tính lại một kết quả đắt đỏ. 200 request = 200 lần query DB giống hệt nhau. Cache càng hiệu quả lúc bình thường, stampede càng thảm khốc lúc miss — vì hệ thống đã được scale theo lượng tải *có* cache.

### Pattern: Single-Flight

Chỉ cho **một** request tính, số còn lại đợi kết quả:

1. Miss → thử lấy **lock theo cache key** (SETNX + TTL ngắn + token ngẫu nhiên).
2. **Winner**: tính → ghi cache → release lock. Release phải **check token** — nếu lock đã hết hạn và người khác chiếm, mình không được xóa lock của họ.
3. **Loser**: không đợi lock, mà **poll chính cache key** trong một cửa sổ ngắn — winner ghi xong là loser đọc được ngay.
4. **Poll timeout → loser tự tính.** Đây là quyết định quan trọng nhất của pattern:

> **Fail-open, không fail-closed.** Winner treo/chết thì hệ thống thoái hóa về hành vi chưa-có-cache (chậm), tuyệt đối không thoái hóa thành lỗi hay response rỗng. Lock trong caching là *tối ưu*, không phải *ràng buộc đúng đắn* — khác hẳn lock chống double-booking.

Hai tham số cần cân theo SLA:
- **Lock TTL** = chặn trên thời gian một winner chết có thể block người khác.
- **Cửa sổ poll** (số lần × khoảng cách) phải nằm gọn trong SLA response của endpoint.

### Ví dụ trong dự án

`reservation-allocator.service.ts` → `computeDayProjection` / `computeDayData`:

```typescript
const lockToken = await this.cache.acquireComputeLock(cacheKey); // SETNX, TTL 3s
if (!lockToken) {
  const awaited = await this.waitForCachedProjection(cacheKey);  // poll 12 × 75ms
  if (awaited) return awaited;                                    // winner ghi kịp
}
try {
  const result = await computeExpensiveThing();                   // winner (hoặc loser timeout) tự tính
  await this.cache.setJson(cacheKey, result);
  return result;
} finally {
  if (lockToken) await this.cache.releaseComputeLock(cacheKey, lockToken);
}
```

Con số cụ thể có lý do: poll 12 × 75ms ≈ 0.9s nằm trong SLA 2 giây của endpoint availability; lock TTL 3s là chặn trên khi winner crash. Muốn đổi phải đo lại theo SLA, không chỉnh tùy tiện.

---

## 5. Lazy Invalidation vs Eager Refresh

### Lý thuyết

Sau khi ghi dữ liệu, có hai cách xử lý cache, và **tên hàm nên phản ánh đúng cách nào**:

| | Lazy invalidation | Eager refresh |
|---|---|---|
| Hành động | Làm cache cũ biến mất / unreachable | Xóa **và** chủ động dựng lại bản mới ngay |
| Ai trả chi phí tính lại | Reader kế tiếp | Writer (hoặc hệ thống phía sau writer) |
| Phù hợp khi | Ghi thường xuyên, nhiều biến thể key, reader chịu được một lần miss chậm | Ghi hiếm, có tầng cache **phía dưới không tự miss được** (trang static, CDN), cần thay đổi hiển thị ngay |

Điểm lý thuyết quan trọng: **cache-aside chỉ tự sửa được các tầng mà reader đi xuyên qua nó**. Nếu phía trước API còn một tầng cache *độc lập* — trang static đã render sẵn (ISR), CDN edge cache — thì xóa cache API xong tầng kia **vẫn serve bản cũ**, vì request không bao giờ chạm tới API để mà miss. Với các tầng đó bắt buộc phải *đẩy* tín hiệu revalidate sang (purge API của CDN, revalidate hook của Next.js...). Đây là lúc cần eager refresh.

Quy tắc đi kèm: bước đẩy sang hệ thống ngoài phải **fail-open** — nó là bước làm-cho-nhanh-hơn, lỗi thì log rồi đi tiếp, không được làm hỏng transaction ghi của writer.

### Ví dụ trong dự án

Hai verb, hai service, đúng theo bảng trên:

- `AllocatorCacheService.invalidateFor*` — **lazy**: chỉ `INCR` version. Booking ghi hàng nghìn lần/ngày, mỗi lần đẻ lại toàn bộ biến thể availability là vô nghĩa — để reader kế tiếp tự tính.
- `WidgetFrontendCacheService.refreshForVenue` — **eager**: xóa key Redis `/config` **rồi** `POST /api/revalidate` sang widget frontend để Next.js render lại trang static ngay. Không có bước 2 thì admin đổi branding xong, guest vẫn thấy HTML cũ đến khi ISR tự hết hạn — vì trang static không bao giờ "miss xuống" API. Bước gọi FE bọc try/catch chỉ log, timeout 5s — fail-open đúng quy tắc.

Tóm gọn cách dự án chia hai verb — invalidate cho dữ liệu đổi liên tục, refresh cho dữ liệu đổi hiếm nhưng phải thấy ngay:

| | `invalidateFor*` (lazy) | `refreshFor*` (eager) |
|---|---|---|
| Cache nào | Availability, pacing, day-data, extras | Payload `/config` + trang static Next.js |
| Làm gì | +1 version counter, **không tính lại gì** — reader sau tự miss rồi tự tính | Xóa key Redis **và** gọi sang FE bắt Next.js render lại trang ngay |
| Ai gọi | Mỗi booking/hold ghi (hàng nghìn lần/ngày) | Admin bấm lưu settings (vài lần/ngày) |
| Vì sao | Ghi quá thường xuyên — tính lại trước cho mọi biến thể là phí; để reader trả chi phí | Trang static **không tự miss được** — không đẩy tín hiệu sang thì cứ serve HTML cũ; admin vừa lưu phải thấy ngay |

---

## 6. Kỷ luật invalidation: blast radius & thứ tự

### Lý thuyết

Hai quy tắc mọi write path phải theo:

**a) Chọn invalidation theo blast radius của write.** Mỗi loại write làm hỏng một phạm vi cache khác nhau — invalidate hẹp hơn phạm vi đó là bug (phantom data), rộng hơn là lãng phí (miss storm không cần thiết). Cách làm bền vững: lập **bảng quy ước write → invalidation** trong doc dự án, write path mới cứ so blast radius mà chọn dòng.

**b) Invalidate SAU khi transaction commit.** Nếu invalidate trước commit:

```
T1: bump version (cache trống)
T2 (reader): miss → đọc DB → thấy dữ liệu CŨ (T1 chưa commit) → cache lại dữ liệu cũ dưới version MỚI
T1: commit
→ cache giữ dữ liệu cũ dưới version mới nhất, không gì đuổi nó đi nữa (trừ TTL)
```

Đây là race kinh điển và khó debug nhất của cache-aside — dữ liệu "thỉnh thoảng sai vài chục giây" không tái hiện được.

### Ví dụ trong dự án

| Write | Gọi gì | Blast radius |
|---|---|---|
| Booking/hold create-update-cancel | `invalidateForDate(venueId, date)` | Availability của đúng ngày đó (booking qua nửa đêm bump cả 2 ngày) |
| Admin sửa services/areas/booking config | `invalidateForVenue` **+** `refreshForVenue` | Availability mọi ngày + payload `/config` + trang static |
| Admin sửa extras | `invalidateExtras(venueId)` | Chỉ catalogue extras, availability không đụng |
| Admin sửa deposit rules / custom fields | `refreshForVenue` (không invalidate allocator) | Chỉ bề mặt config |

Mọi call site đều đứng sau khi DB write hoàn tất — đúng quy tắc (b).

---

## 7. Biết cái gì KHÔNG cache

### Lý thuyết

Cache key dùng chung chỉ được chứa kết quả **thuần túy là hàm của key**. Nếu kết quả còn phụ thuộc tham số per-request/per-user không nằm trong key, cache bản đó ra là **cache poisoning tự gây**: user A nhận response tính riêng cho user B. Hai lối thoát: đưa tham số đó vào key (nếu số biến thể hữu hạn và đáng cache), hoặc **bypass cache hẳn** cho nhánh đó (nếu biến thể ~ vô hạn, mỗi biến thể chỉ 1 người đọc — cache hit rate ≈ 0, cache chỉ còn rủi ro).

Ngoài ra: khi kết quả đắt là một aggregate, cân nhắc **cache con số tổng hợp thay vì dữ liệu thô** — rẻ hơn nhiều về bộ nhớ và serialize, cùng cơ chế invalidate.

### Ví dụ trong dự án

- Guest sửa booking của mình → availability tính với `excludeReservation` (trừ chính booking đang sửa ra). Kết quả này đúng cho *duy nhất* booking đó → `booking-widget.service.ts` **bypass cache hoàn toàn** ở nhánh này, comment ghi rõ lý do.
- `pacing-counter.service.ts` cache **một con số** (tổng covers đang active của một service) chứ không cache danh sách booking — aggregate-not-rows.

---

## 8. Checklist áp dụng cho dự án khác

Thiết kế cache cho một read path mới, đi qua các câu hỏi này theo thứ tự:

1. **Có đáng cache không?** Đo trước: tần suất đọc × chi phí tính. Đọc hiếm hoặc tính rẻ → đừng cache, cache là complexity có lãi suất.
2. **Dữ liệu thuộc loại nào?** Config-shaped (TTL dài + refresh khi ghi) hay state-shaped (TTL ngắn làm backstop + invalidate chủ động ở *mọi* write path)?
3. **Key có bao nhiêu biến thể trên một thực thể?** Nhiều → version-keyed invalidation ngay từ đầu, đừng để phải refactor khỏi SCAN+DEL như dự án này đã phải làm.
4. **Blast radius của từng write?** Lập bảng write → invalidation, chọn granularity của version counter theo bảng (cần mấy tầng? stride bao nhiêu?).
5. **Key có hot không?** Nhiều reader đồng thời trên cùng key → single-flight lock, tham số poll/TTL cân theo SLA, và luôn fail-open.
6. **Còn tầng cache nào phía trước mà reader không đi xuyên qua?** (static page, CDN) → cần eager refresh đẩy sang, fail-open.
7. **Kết quả có phụ thuộc gì ngoài key không?** Có → đưa vào key hoặc bypass, tuyệt đối không nhét vào key dùng chung.
8. **Invalidate đặt sau commit chưa?** Kiểm tra từng call site.
9. **Redis chết thì sao?** Mọi đường phải degrade về chậm-nhưng-đúng. Nếu có đường nào degrade về sai hoặc lỗi — cache đang là source of truth, thiết kế lại.

---

## 9. Các chiến lược caching khác (tham khảo nhanh)

Doc này xoay quanh **cache-aside** vì nó phổ biến nhất; dưới đây là các chiến lược còn lại và khi nào chúng thắng.

| Chiến lược | Cách hoạt động | Dùng khi | Đánh đổi |
|---|---|---|---|
| **Read-Through** | Như cache-aside nhưng logic load nằm trong *tầng cache* (library/proxy tự query DB khi miss), app chỉ gọi `cache.get()` | Nhiều nơi cùng đọc một loại dữ liệu — tránh lặp code load | Cần cache layer thông minh; ít kiểm soát cách query |
| **Write-Through** | Mọi write đi *xuyên qua* cache: ghi cache + ghi DB đồng bộ trong cùng thao tác | Đọc ngay sau ghi phải hit; chấp nhận write chậm hơn | Write latency tăng; cache chứa cả dữ liệu không ai đọc |
| **Write-Behind (Write-Back)** | Ghi vào cache trước, trả về ngay; DB được ghi *bất đồng bộ* sau (batch) | Write cực nhiều, chịu được mất dữ liệu vài giây khi cache chết (counter, analytics, view count) | **Rủi ro mất dữ liệu** — cache tạm thời là source of truth; phức tạp nhất |
| **Refresh-Ahead** | Cache tự làm mới key hot *trước khi* TTL hết, dựa trên tần suất truy cập | Key rất hot + tính lại đắt + không chịu nổi latency spike lúc miss | Dự đoán sai key hot = tính lại vô ích; thêm background machinery |
| **Stale-While-Revalidate** | Hết TTL vẫn trả bản cũ ngay, đồng thời kích tính lại ở background cho lần sau | Latency quan trọng hơn độ tươi vài giây (HTTP caching, CDN có sẵn cơ chế này) | Có cửa sổ stale có chủ đích; cần định nghĩa "cũ tối đa bao nhiêu" |
| **Negative Caching** | Cache cả kết quả "không tồn tại" (404, empty) với TTL ngắn | Bị hammer bởi lookup thứ không tồn tại (chống cache-penetration, ID rác) | Tạo xong dữ liệu thật phải nhớ invalidate bản negative |
| **L1/L2 (in-memory + Redis)** | Cache 2 tầng: in-process (nanosecond) trước, Redis sau | Key cực hot mà round-trip Redis (~1ms) vẫn là bottleneck | L1 của mỗi instance stale độc lập — cần TTL rất ngắn hoặc pub/sub invalidate |
| **Probabilistic Early Expiration** | Mỗi reader *xác suất* tự tính lại sớm trước TTL (xác suất tăng dần khi gần hết hạn) | Chống stampede không cần lock — thay thế single-flight khi không muốn cơ chế lock/poll | Vẫn có xác suất nhỏ nhiều reader cùng tính; khó reason hơn lock |

### 9a. Read-Through

- **Ý tưởng dễ hiểu:** cache-aside nhưng app "lười" hơn — app chỉ hỏi cache, còn *cache tự biết* cách đi lấy dữ liệu khi miss. Logic "miss thì query DB" viết một lần trong cache layer thay vì lặp lại ở mọi service.
- **Ví dụ đơn giản:** ORM có second-level cache (Hibernate), hoặc tự viết:

```typescript
// Thay vì mỗi service tự viết if-miss-then-query:
const user = await cache.getOrLoad(`user:${id}`, () => userRepo.findById(id), 300);
// getOrLoad gói toàn bộ: get → miss → gọi loader → set TTL → trả về
```

### 9b. Write-Through

- **Ý tưởng dễ hiểu:** ghi tiền vào sổ ngân hàng *và* ví cùng lúc — đọc ví lúc nào cũng khớp sổ. Mọi write cập nhật cache ngay trong cùng thao tác, nên không bao giờ có cửa sổ stale sau khi ghi.
- **Ví dụ đơn giản:** profile user mà màn hình ngay sau khi bấm "Lưu" phải hiện đúng:

```typescript
async updateProfile(id, dto) {
  const user = await this.userRepo.update(id, dto);
  await cache.set(`user:${id}`, user, 3600); // ghi đè cache luôn, không đợi reader
  return user;
}
```

### 9c. Write-Behind (Write-Back)

- **Ý tưởng dễ hiểu:** ghi nháp vào sổ tay trước, cuối ngày mới chép vào sổ cái. Write trả về ngay vì chỉ chạm cache; DB được gom lại ghi sau theo lô. Nhanh nhất, nhưng sổ tay mất trước khi chép = mất dữ liệu.
- **Ví dụ đơn giản:** đếm view bài viết — 10.000 view/phút mà mỗi view một UPDATE thì DB chết:

```typescript
await redis.incr(`views:${postId}`);              // mỗi view chỉ chạm Redis
// Cron mỗi 60s: đọc counter → UPDATE posts SET views = views + n → reset counter
// Redis sập giữa chừng → mất tối đa 60s view count: chấp nhận được với số liệu này
```

### 9d. Refresh-Ahead

- **Ý tưởng dễ hiểu:** quán cơm thấy món hot sắp hết thì nấu nồi mới *trước khi* nồi cũ cạn — khách không bao giờ phải đứng đợi. Cache tự làm mới key hot trước khi TTL hết, reader không bao giờ dính miss chậm.
- **Ví dụ đơn giản:** bảng xếp hạng trang chủ, query mất 3s, TTL 5 phút:

```typescript
// Cron chạy mỗi 4 phút (trước khi TTL 5 phút hết):
const board = await computeLeaderboard();          // 3s, nhưng chạy nền, không ai đợi
await cache.set('leaderboard', board, 300);
// Reader luôn hit — latency spike 3s biến mất khỏi request path
```

### 9e. Stale-While-Revalidate

- **Ý tưởng dễ hiểu:** báo hết hạn vẫn đưa khách đọc tạm số cũ, đồng thời sai người đi lấy số mới cho khách sau. Không ai phải đợi, đổi lại có người đọc tin cũ vài giây.
- **Ví dụ đơn giản:** HTTP caching có sẵn cú pháp cho nó:

```
Cache-Control: max-age=60, stale-while-revalidate=300
# 0–60s:    trả cache, tươi
# 60–360s:  vẫn trả bản cũ NGAY, đồng thời fetch bản mới ở background
# >360s:    bắt buộc đợi fetch mới
```

### 9f. Negative Caching

- **Ý tưởng dễ hiểu:** khách hỏi món không có trong menu, thay vì lần nào cũng chạy xuống bếp kiểm tra, ghi luôn ra giấy "món này KHÔNG có" dán trước quầy. Cache cả câu trả lời "không tồn tại".
- **Ví dụ đơn giản:** chống bot quét API bằng ID rác — mỗi request rác là một query DB vô ích:

```typescript
const cached = await cache.get(`user:${id}`);
if (cached === 'NOT_FOUND') throw new NotFoundException(); // chặn từ cache
if (cached) return JSON.parse(cached);
const user = await this.userRepo.findById(id);
if (!user) {
  await cache.set(`user:${id}`, 'NOT_FOUND', 60);          // TTL ngắn thôi
  throw new NotFoundException();
}
```

### 9g. L1/L2 (in-memory + Redis)

- **Ý tưởng dễ hiểu:** giấy nhớ trên bàn (L1 — với tay là lấy) và tủ hồ sơ cuối phòng (L2 — phải đứng dậy). Thứ tra liên tục thì chép ra giấy nhớ, chấp nhận giấy nhớ có thể lỗi thời hơn tủ một chút.
- **Ví dụ đơn giản:** feature flags đọc ở *mọi* request — 1ms round-trip Redis × mọi request vẫn là đáng kể:

```typescript
const l1 = new Map();                               // trong process, TTL 5s
async getFlag(name) {
  const hit = l1.get(name);
  if (hit && hit.expires > monotonicNow()) return hit.value;
  const value = await redis.get(`flag:${name}`);    // L2, TTL dài hơn
  l1.set(name, { value, expires: monotonicNow() + 5000 });
  return value;
}
// Mỗi instance có L1 riêng → flag đổi thì instance này thấy sau instance kia tối đa 5s
```

### 9h. Probabilistic Early Expiration

- **Ý tưởng dễ hiểu:** thay vì cả phòng cùng ùa đi lấy nước lúc bình cạn, mỗi người *tung xúc xắc* — bình càng gần cạn, xác suất "tôi đi lấy luôn" càng cao. Thường chỉ một người đi sớm, những người còn lại vẫn có nước uống, và khoảnh khắc bình-cạn-cả-phòng-ùa-đi không bao giờ xảy ra.
- **Ví dụ đơn giản:**

```typescript
const { value, savedAt, ttl, computeMs } = await cache.getWithMeta(key);
// Càng gần hết hạn (và tính càng đắt), càng dễ "trúng số" tự tính lại sớm:
const shouldRecomputeEarly =
  value && monotonicNow() > savedAt + ttl - computeMs * beta * -Math.log(random());
if (!value || shouldRecomputeEarly) {
  return recomputeAndSet(key);                      // một reader lẻ tự làm mới sớm
}
return value;                                       // số còn lại dùng bản cũ vẫn hợp lệ
```

Cách chọn nhanh:
- **Đọc nhiều, ghi ít** → cache-aside (mặc định) hoặc read-through.
- **Đọc ngay sau ghi phải đúng** → write-through.
- **Ghi dồn dập, chịu được mất mát** → write-behind.
- **Sợ latency spike lúc miss** → single-flight (mục 4), refresh-ahead, hoặc stale-while-revalidate — ba giải pháp cho cùng một vấn đề, khác nhau ở chỗ ai trả chi phí và stale bao lâu.
- **Bị dội bởi key không tồn tại** → negative caching.

Lưu ý cuối: các chiến lược write-path (write-through/behind) làm cache **tham gia vào đường ghi** — trái với nguyên tắc "cache chết thì hệ thống chỉ chậm đi" của cache-aside. Chỉ chọn khi đo được lợi ích rõ ràng, và hiểu rằng bạn đang nâng cache từ tầng tăng tốc lên một phần của source of truth.
