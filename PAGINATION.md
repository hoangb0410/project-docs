# PAGINATION: OFFSET, KEYSET/CURSOR VÀ THIẾT KẾ API PHÂN TRANG

Lý thuyết chung về phân trang cho backend — áp dụng được cho mọi dự án có bảng lớn; nollie-api xuất hiện ở cuối mỗi phần như ví dụ thực tế (dự án có sẵn cả ba mô hình: offset ở danh sách reservation, keyset ở timeline, cursor loop ở export khách hàng).

---

## 1. Vì sao phải phân trang

Một truy vấn không giới hạn (`findAll` không `limit`) có ba đường chết, và cả ba đều **không xuất hiện lúc dev** vì bảng dev nhỏ:

1. **RAM của app** — ORM hydrate toàn bộ kết quả thành object trước khi trả; 1 triệu bản ghi là đủ giết một instance Node.
2. **DB** — đọc và truyền toàn bộ bảng, chiếm connection lâu, đẩy các truy vấn khác vào hàng đợi.
3. **Payload** — response hàng chục MB làm client (mobile, browser) nghẹn, timeout ở proxy.

Vì lỗi chỉ phát nổ ở production khi dữ liệu đã lớn, phân trang phải là **luật mặc định khi viết code**, không phải tối ưu thêm sau.

> **Trong dự án:** luật cứng trong development rules — mọi query bảng lớn phải paginate, default limit 100, hard cap 1.000; `RESERVATION_LIST_MAX_LIMIT = 500` cho các list đặt bàn.

---

## 2. Nguyên tắc nền: thứ tự phải deterministic

Trước khi chọn mô hình nào, phải hiểu điều kiện tiên quyết của *mọi* mô hình. Phân trang bản chất là **cắt một dãy thành nhiều lát, lấy qua nhiều request cách nhau về thời gian**. Toàn bộ cơ chế chỉ đúng khi mọi request đều nhìn thấy **cùng một dãy, cùng một thứ tự** — tính deterministic. Thứ tự mà đổi giữa hai request thì các lát không ghép lại thành dãy gốc: bản ghi lặp hoặc biến mất tại ranh giới trang, bất kể offset hay keyset.

Ba nguồn phá vỡ deterministic, xếp theo mức phổ biến:

### 2.1. Không có `ORDER BY`

SQL không cam kết thứ tự nào khi thiếu `ORDER BY` — Postgres trả theo kế hoạch thực thi (seq scan, index scan, parallel worker nào xong trước), và kế hoạch đổi theo statistics, theo tải, theo từng lần chạy. Hai request liên tiếp có thể nhận hai hoán vị khác nhau của cùng dữ liệu → `LIMIT/OFFSET` cắt trên hai dãy khác nhau, kết quả vô nghĩa. **Endpoint phân trang không có `ORDER BY` là bug, kể cả khi "chạy thấy đúng"** — nó chỉ đúng cho đến khi planner đổi kế hoạch.

### 2.2. `ORDER BY` theo cột không unique — thiếu tie-breaker

Sort theo `created_at` khi nhiều bản ghi cùng giá trị (import batch, seed data): DB xếp các dòng bằng nhau **tùy ý**, mỗi lần chạy một kiểu. Lỗi phát nổ đúng tại ranh giới trang:

```
Thứ tự lần gọi 1:  ... | A B [hết trang 1]   C D ...     (A,B,C,D cùng created_at)
Thứ tự lần gọi 2:  ... | A C [đã lấy rồi]    B D ...
→ trang 2 trả C D: bản ghi B biến mất, không ai biết
```

Fix bắt buộc: **tie-breaker bằng cột unique**, thường là khóa chính:

```sql
ORDER BY created_at DESC, id DESC
```

Với keyset, điều kiện cursor cũng phải thành so sánh bộ tương ứng: `WHERE (created_at, id) < ($lastCreatedAt, $lastId)`. Cùng họ vấn đề: cột sort chứa `NULL` phải chốt tường minh `NULLS FIRST/LAST` — mặc định khác nhau giữa các DB và giữa ASC/DESC.

### 2.3. Khóa sort bị update giữa chừng

Sort theo cột mutable (`updated_at`, `score`, `status`): thứ tự deterministic *tại một thời điểm*, nhưng bản ghi bị update sẽ **nhảy vị trí qua ranh giới cursor** giữa hai lần gọi — bản ghi đã đọc quay lại phía trước cursor (đọc lặp), hoặc bản ghi chưa đọc nhảy ra sau lưng cursor (mất hẳn). Đây là dạng khó thấy nhất vì từng query nhìn riêng đều "đúng".

Hệ quả thực dụng: **vòng lặp máy-đọc-máy (export, sync, batch) phải sort theo khóa immutable** — `id` tự tăng hoặc `created_at` — không bao giờ theo cột nghiệp vụ có thể đổi trong lúc job chạy.

> **Trong dự án:** cả hai đường keyset đều đã chọn khóa đúng chuẩn này — export khách hàng lặp theo `id` (immutable, unique, tự nó là tie-breaker), timeline phân trang theo `tableId`. Không có đường phân trang nào sort theo cột mutable.

---

## 3. Ba mô hình phân trang

### 3.1. Offset pagination — "trang số N"

Client gửi `limit` + `offset` (hoặc `page`), server dịch thẳng sang SQL:

```sql
SELECT * FROM bookings WHERE venue_id = $1
ORDER BY created_at DESC, id DESC
LIMIT 50 OFFSET 200;   -- trang 5, mỗi trang 50
```

**Cách DB thực thi — và điểm yếu chí mạng:** `OFFSET 200` không có nghĩa là "nhảy đến dòng 201". DB vẫn phải **đọc đủ 250 dòng theo thứ tự sort rồi vứt 200 dòng đầu**. Chi phí tuyến tính theo độ sâu trang: trang 1 nhanh, trang 1.000 (`OFFSET 50000`) đọc bỏ 50.000 dòng — càng lật sâu càng chậm, và đây là gậy để kẻ xấu đánh DB qua public API.

**Điểm yếu thứ hai — trôi trang (page drift):** offset neo trang vào **vị trí**, mà vị trí thì xê dịch khi dữ liệu đổi. Giữa hai lần gọi, một bản ghi mới chèn vào đầu danh sách làm mọi offset lệch đi 1 → trang sau **lặp lại** bản ghi cuối trang trước; một bản ghi bị xóa thì **nuốt mất** một bản ghi. Lưu ý đây là hạn chế *cố hữu của mô hình*, xảy ra ngay cả khi thứ tự đã deterministic hoàn hảo theo mục 2 — deterministic là điều kiện cần, không phải đủ, với offset.

**Đổi lại, offset có hai thứ keyset không có:** nhảy thẳng đến trang bất kỳ (trang 7 không cần lật qua trang 1–6), và dễ hiển thị "trang 3/12" (kèm một `COUNT` riêng).

**Dùng khi:** UI dạng bảng có ô số trang, dữ liệu tương đối tĩnh, và độ sâu trang thực tế nông (người dùng hiếm khi lật quá vài chục trang).

> **Trong dự án:** `GET /reservations` (kanban) dùng `limit` + `offset` — `list-reservations.query.dto.ts` với `@Min(0)` cho offset, `@Max(500)` cho limit. Hợp lý vì phạm vi đã bị chặn theo venue + ngày + service nên tổng số dòng nhỏ, độ sâu trang không bao giờ lớn.

### 3.2. Keyset (cursor) pagination — "tiếp sau bản ghi X"

Thay vì đếm-bỏ N dòng, client gửi **giá trị khóa của bản ghi cuối cùng đã thấy**, server seek thẳng vào index:

```sql
SELECT * FROM customers WHERE venue_id = $1
  AND id > $lastId          -- keyset: tiếp sau bản ghi cuối trang trước
ORDER BY id
LIMIT 100;
```

**Vì sao nhanh ổn định:** `id > $lastId` là một cú seek B-tree — O(log n) bất kể đang ở "trang" thứ mấy. Trang 1 và trang 10.000 tốn như nhau.

**Vì sao không trôi:** trang được neo vào *giá trị*, không phải *vị trí*. Bản ghi chèn/xóa phía trước cursor không ảnh hưởng gì đến các trang sau.

**Hai điều kiện bắt buộc:**

1. **Khóa sort thỏa mục 2** — unique (hoặc có tie-breaker, dùng so sánh bộ) và immutable. Keyset *nhạy* với vi phạm deterministic hơn offset: cursor là một giá trị của khóa sort, khóa mà trùng hoặc đổi thì cursor mất chỗ neo.
2. **Có index khớp đúng thứ tự sort** — keyset không có index là scan trá hình.

**Cái giá phải trả:** không nhảy trang tùy ý (chỉ đi tuần tự tới/lui), không có "trang 3/12" tự nhiên.

**Dùng khi:** infinite scroll / feed, dataset lớn, dữ liệu ghi liên tục — và **bắt buộc** cho vòng lặp máy-đọc-máy (export, sync, batch job): job chạy hàng giờ trên dữ liệu đang được ghi thì offset vừa chậm dần vừa trôi, keyset là lựa chọn duy nhất đúng.

> **Trong dự án — hai ví dụ:**
> - **UI:** timeline đặt bàn phân trang dọc theo bàn — `afterTableId` trong `timeline-blocks.query.dto.ts` là cursor (= `pagination.nextAfterTableId` của response trước), client lật đến khi `hasMore === false`.
> - **Batch:** `getMatchingCustomerIdsBatch` trong `customer.service.ts` — vòng lặp `id: { [Op.gt]: lastId }` + `ORDER BY id` + batch size, bắt đầu từ `lastId = 0`, dùng cho streaming insert danh sách khách hàng lớn để không nổ RAM. Development rules cũng chốt: thao tác tuần tự lớn (customer sync, export) phải cursor-based.

### 3.3. Opaque cursor — keyset giấu sau một token

Bản chất vẫn là keyset, nhưng thay vì phơi `lastId`/`lastCreatedAt` ra query param, server **encode trạng thái cursor thành token mờ** (thường base64 của JSON, có thể ký):

```
GET /v1/charges?limit=100&starting_after=ch_3Nx...   ← kiểu Stripe
GET /items?cursor=eyJpZCI6MTIzNCwiY3JlYXRlZCI6...    ← cursor tự đóng gói
```

**Được gì so với keyset trần:** contract của API chỉ còn "trả cursor này lại cho tôi" — client không biết và không được dựa vào cấu trúc bên trong, nên server **đổi chiến lược sort/khóa mà không vỡ API**; đồng thời chặn client tự chế cursor bậy (nhất là khi token có ký).

**Giá phải trả:** thêm một tầng encode/decode + validate token (hết hạn, sai định dạng → 400 rõ ràng, đừng 500).

**Dùng khi:** API public cho bên thứ ba — contract sống lâu hơn schema. API nội bộ giữa FE–BE của cùng một team thì keyset trần (`afterTableId`) đơn giản hơn và đủ tốt.

> **Trong dự án:** chưa tự phát hành opaque cursor, nhưng *tiêu thụ* nó hàng ngày: Stripe list API (`starting_after` + `has_more`), và các POS API trả `NextUrl` — đúng ràng buộc đã ghi trong `STREAMING-SYNC-ARCHITECTURE.md`: nguồn chỉ cấp cursor thì không fan-out song song được, phải lật tuần tự.

### 3.4. Bảng so sánh ba mô hình

| | Offset | Keyset trần | Opaque cursor |
| --- | --- | --- | --- |
| Chi phí trang sâu | Tuyến tính — đọc bỏ N dòng | O(log n) — như nhau mọi trang | Như keyset |
| Ổn định khi dữ liệu ghi liên tục | ❌ Trôi trang (lặp/mất bản ghi) | ✅ Neo theo giá trị | ✅ Như keyset |
| Nhảy đến trang bất kỳ | ✅ | ❌ Chỉ tuần tự tới/lui | ❌ |
| Hiển thị "trang 3/12" / total | ✅ (kèm COUNT riêng) | ❌ Không tự nhiên | ❌ |
| Độ phức tạp cài đặt | Thấp nhất | Trung bình (cursor + tie-breaker + index) | Cao nhất (+ encode/validate token) |
| Ràng buộc schema | Chỉ cần ORDER BY deterministic | Khóa sort unique + immutable + index khớp | Như keyset |
| Che giấu nội bộ / đổi impl không vỡ contract | ❌ | ❌ (client thấy khóa sort) | ✅ |
| Hợp với | UI trang số, dữ liệu tĩnh, trang nông | Infinite scroll, batch/export | API public cho bên thứ ba |

### 3.5. Keyset nâng cao — nhiều cột sort, filter và search

Ba câu hỏi thực tế luôn gặp khi rời khỏi ví dụ `ORDER BY id`:

**a) Sort nhiều cột → composite cursor.** Cursor phải chứa **đủ giá trị của mọi cột sort** (kể cả tie-breaker), không chỉ id. Với các cột **cùng chiều**, Postgres có row-value comparison ánh xạ thẳng và tận dụng được index composite:

```sql
-- ORDER BY created_at DESC, id DESC
WHERE (created_at, id) < ($lastCreatedAt, $lastId)
ORDER BY created_at DESC, id DESC
LIMIT 100;
```

Với sort **trộn chiều** (`created_at DESC, name ASC`), row-value comparison không dùng được — phải khai triển thành chuỗi OR lồng, mỗi tầng "đóng băng" các cột trước và so cột kế:

```sql
WHERE created_at < $1
   OR (created_at = $1 AND name > $2)
   OR (created_at = $1 AND name = $2 AND id > $3)
```

và index phải khai báo đúng chiều từng cột: `(created_at DESC, name ASC, id ASC)`. Sequelize không có row-value comparison sẵn — viết `sequelize.literal` cho điều kiện này, hoặc dạng `Op.or` khai triển. Càng nhiều cột sort, cursor và điều kiện càng phình — đây là lý do thực dụng để **giới hạn số kiểu sort mà API public cho phép**, thay vì "sort theo cột nào cũng được".

**b) Filter/search text đi kèm → cursor gắn chặt với bộ filter.** Filter không phá keyset — nó chỉ thu hẹp dãy, cursor vẫn seek bình thường *miễn là mọi trang dùng đúng một bộ filter*. Điều vỡ là **đổi filter giữa chừng**: bộ filter khác = dãy khác = cursor cũ trỏ vào một vị trí vô nghĩa. Ba hệ quả:

- Client **phải reset cursor về đầu khi filter đổi** — đây là contract, cần ghi rõ trong API doc.
- Opaque cursor giải quyết đẹp hơn: **nhúng hash của bộ filter vào token**; server nhận cursor kèm filter không khớp hash → trả 400 rõ ràng thay vì lặng lẽ trả kết quả sai.
- Index phải phục vụ cả filter lẫn sort: cột filter đẳng thức đứng trước, cột sort đứng sau trong composite index (`(venue_id, created_at DESC, id)`) — thiếu thì mỗi trang là một lần scan + sort lại.

> **Trong dự án:** timeline làm đúng contract này — FE đổi service filter thì reset pagination rồi gọi lại `timeline/blocks` với `serviceId` mới, cursor `afterTableId` không mang qua (đã chốt trong `docs/reservation-timeline-fe.md`).

**c) Search theo độ liên quan (relevance) → keyset khó sống.** `ORDER BY rank` với rank là giá trị *tính toán* (full-text score): không unique, và có thể đổi giữa hai lần chạy khi index/dữ liệu đổi — vi phạm cả hai điều kiện của mục 2. Hai lối ra thực dụng:

- **UI search cho người**: offset nông + cap chặt số trang. Người dùng hiếm khi lật quá vài trang kết quả search (Google cũng cắt ở ~vài chục trang) — chi phí offset không kịp thành vấn đề.
- **Search engine chuyên dụng**: Elasticsearch có `search_after` = chính là keyset trên `(score, tie-breaker)` — cùng nguyên tắc composite cursor ở (a), engine lo phần seek.

---

## 4. Thiết kế response envelope

Mọi endpoint phân trang nên trả cùng một hình dạng:

```jsonc
{
  "items": [...],
  "pagination": {
    "hasMore": true,
    "nextCursor": "eyJ..."   // hoặc nextAfterTableId, tùy mô hình
  }
}
```

Ba quyết định thiết kế đáng lưu ý:

- **`hasMore` rẻ nhất bằng trick `limit + 1`:** query `limit + 1` bản ghi; nếu nhận đủ `limit + 1` thì `hasMore = true` và cắt bỏ bản ghi thừa. Không tốn thêm query nào.
- **Đừng trả `total` mặc định.** `COUNT(*)` trên bảng lớn là một full scan đắt ngang chính query — và bị trả *mỗi lần lật trang*. Chỉ trả total khi UI thật sự cần ("trang 3/12"), tách thành query/endpoint riêng, hoặc dùng số ước lượng từ statistics của DB nếu chỉ cần hiển thị "~12.000 kết quả".
- **Cursor nằm trong response, không để client tự suy.** Client cầm `nextCursor` server đưa và trả lại nguyên văn — client tự lấy `items[items.length - 1].id` làm cursor là contract ngầm, vỡ ngay khi server đổi khóa sort. (Đây là lý do timeline trả `nextAfterTableId` tường minh thay vì để FE lấy id bàn cuối.)

---

## 5. Các quy tắc an toàn đi kèm

Phân trang đúng mô hình vẫn hỏng nếu thiếu bốn thứ này:

1. **Default + hard cap ngay trong DTO/validation layer** — client không gửi `limit` thì server tự đặt (không bao giờ hiểu là "lấy hết"); client gửi `limit=999999` thì bị chặn từ cửa. Cap là một phần của query cost limiting (xem `PUBLIC-API-PROTECTION.md` mục 4.1), đặc biệt bắt buộc trên public API.

```typescript
@IsOptional()
@Min(1)
@Max(RESERVATION_LIST_MAX_LIMIT)   // hard cap = hằng số, không rải magic number
limit?: number;                     // service tự default khi undefined
```

2. **Thứ tự deterministic** — toàn bộ mục 2: có `ORDER BY`, tie-breaker cho cột không unique, khóa immutable cho vòng máy-đọc-máy.
3. **Index khớp thứ tự sort** — cả offset lẫn keyset đều dựa vào index để tránh sort toàn bảng; keyset composite cần index composite cùng chiều.
4. **Scope trước, paginate sau** — phân trang không thay thế điều kiện lọc: luôn thu hẹp phạm vi (theo tenant, theo ngày) rồi mới phân trang phần còn lại. Cap 500 của reservation list "đủ" vì query đã scope theo venue + ngày; cùng cap đó trên query không scope là vô dụng.

---

## 6. Chọn mô hình nào — bảng quyết định

| Tình huống | Mô hình | Lý do |
| --- | --- | --- |
| UI bảng có ô số trang, dữ liệu ít ghi, trang nông | Offset | Nhảy trang tùy ý, code đơn giản nhất |
| Infinite scroll / feed, dữ liệu ghi liên tục | Keyset | Không trôi trang, nhanh ổn định mọi độ sâu |
| Export / sync / batch job trên bảng lớn | Keyset (bắt buộc) | Job dài trên dữ liệu đang ghi: offset vừa chậm dần vừa trôi |
| API public cho bên thứ ba | Opaque cursor | Che nội bộ, đổi implementation không vỡ contract |
| Bảng nhỏ đã scope chặt (< vài nghìn dòng) | Offset + cap | Chi phí offset không đáng kể, khỏi phức tạp hóa |

Quy tắc nhớ nhanh: **người lật trang → offset còn chấp nhận được; máy lật trang → luôn là keyset.**

---

## 7. Checklist khi thêm một endpoint trả danh sách

1. Có `limit` với default + hard cap trong DTO? (không bao giờ để "không gửi = lấy hết")
2. Thứ tự deterministic (mục 2): có `ORDER BY`, tie-breaker nếu cột sort không unique, khóa immutable nếu là vòng lặp batch?
3. Chọn mô hình theo bảng ở mục 6 — client là người hay máy?
4. Keyset: có index khớp thứ tự sort? Cursor do server trả trong response, không để client tự suy? Sort nhiều cột thì cursor chứa đủ mọi cột (mục 3.5a)?
5. Có filter đi kèm: contract "đổi filter = reset cursor" đã ghi rõ (mục 3.5b)?
6. Response có `hasMore` (trick `limit + 1`)? `total` chỉ trả khi UI thật cần và tách riêng?
7. Đã scope (tenant/ngày) trước khi phân trang?
