# Concurrency Control — Lý thuyết và cách nollie-api áp dụng

> Tài liệu tham khảo nội bộ. Phần 1–2 là lý thuyết thuần, phần 3 map vào codebase, phần 4 là các bẫy thực tế.

---

## 1. Vấn đề gốc: race condition

Hai request chạy song song cùng đọc – rồi cùng ghi một dữ liệu. Không có cơ chế bảo vệ thì xảy ra:

| Tên                               | Kịch bản                                                | Ví dụ trong domain                                   |
| --------------------------------- | ------------------------------------------------------- | ---------------------------------------------------- |
| **Lost update**                   | A đọc, B đọc, A ghi, B ghi đè → thay đổi của A biến mất | 2 staff cùng sửa 1 reservation trên kanban           |
| **Double spend / double booking** | Cả hai cùng thấy "còn chỗ" rồi cùng ghi                 | 2 khách widget cùng giành 1 bàn cuối cùng trong slot |
| **Read-modify-write sai**         | Đọc counter = 5, cả hai cùng ghi 6 (đáng lẽ 7)          | Đếm quota `venue_usages`                             |

Mọi cơ chế lock đều trả lời một câu hỏi: **chặn xung đột trước khi nó xảy ra, hay phát hiện nó sau khi xảy ra?** Đó chính là ranh giới pessimistic vs optimistic.

---

## 2. Lý thuyết

### 2.1. Pessimistic locking — "khóa trước, làm sau"

**Giả định:** xung đột _chắc chắn hoặc thường xuyên_ xảy ra → giành quyền độc quyền ngay khi đọc.

**Cơ chế trong PostgreSQL:**

```sql
BEGIN;
SELECT * FROM tables WHERE id = 42 FOR UPDATE;  -- khóa row
-- transaction khác chạy FOR UPDATE trên row 42 sẽ BLOCK tại đây, xếp hàng chờ
UPDATE tables SET ... WHERE id = 42;
COMMIT;  -- lock nhả ra, transaction đang chờ được chạy tiếp
```

Điểm mấu chốt: sau khi transaction đứng chờ được đánh thức, nó **đọc lại row ở trạng thái mới nhất** (đã bao gồm thay đổi của transaction trước). Vì vậy pattern chuẩn luôn là: _lock → đọc lại/kiểm tra điều kiện → ghi_ — kiểm tra điều kiện phải nằm **sau** khi lock, không phải trước.

**Các biến thể:**

- `FOR UPDATE` — khóa độc quyền để ghi. Mặc định dùng cái này.
- `FOR SHARE` — nhiều reader cùng giữ được, chặn writer. Ít dùng.
- `FOR UPDATE NOWAIT` — không chờ, lỗi ngay nếu row đang bị khóa.
- `FOR UPDATE SKIP LOCKED` — bỏ qua row đang bị khóa (pattern job-queue trên DB).

**Advisory lock** — khóa pessimistic nhưng **không gắn vào row nào**, khóa theo một con số/key logic do ứng dụng tự đặt:

```sql
SELECT pg_advisory_xact_lock(hashtext('pos-party:venue-9:2026-08-17'));
```

Dùng khi thứ cần serialize không phải là một row có sẵn — ví dụ "mọi xử lý cho cùng một party POS" hay "chỉ một tiến trình import cho venue này" — kể cả khi row _chưa tồn tại_ (chống double-insert). Biến thể `_xact_` tự nhả khi transaction kết thúc, không lo quên unlock.

**Ưu / nhược của pessimistic:**

- ✅ Đúng tuyệt đối: xung đột không bao giờ xảy ra, không cần retry.
- ✅ Người đến sau _chờ_ rồi vẫn được xử lý (queue), không bị reject.
- ❌ Giữ connection + row trong suốt thời gian lock → không được làm việc chậm (gọi API ngoài, sleep) khi đang giữ lock.
- ❌ Nguy cơ **deadlock**: A khóa row 1 chờ row 2, B khóa row 2 chờ row 1. Postgres tự phát hiện và kill một bên, nhưng phòng bằng cách **luôn khóa theo thứ tự cố định** (vd. sort id tăng dần trước khi lock).

### 2.2. Optimistic locking — "làm trước, kiểm tra sau"

**Giả định:** xung đột _hiếm_ → không khóa gì cả, chỉ mang theo bằng chứng về phiên bản đã đọc, lúc ghi so lại.

**Cơ chế:** mỗi bản ghi có một "version" — số tăng dần, hoặc chính `updated_at`. Flow:

```
1. Client đọc record, giữ lại version V (vd. updatedAt = "10:00:05.123")
2. Client submit thay đổi kèm V
3. Server so V với version hiện tại trong DB:
   - khớp  → ghi, version tự nhảy
   - lệch  → có người sửa trong lúc đó → reject 409/412, client reload rồi thử lại
```

Dạng thuần SQL (compare-and-swap ngay trong câu UPDATE, atomic không cần transaction dài):

```sql
UPDATE reservations SET status = 'SEATED', version = version + 1
WHERE id = 42 AND version = 7;
-- rowCount = 0 nghĩa là version đã lệch → conflict
```

**Ưu / nhược:**

- ✅ Không giữ lock → phù hợp "transaction" kéo dài theo thao tác người dùng (mở form sửa vài phút — không thể giữ row lock lâu vậy).
- ✅ Không deadlock, không chặn ai, throughput đọc cao.
- ❌ Người thua bị **reject** chứ không được xếp hàng → phải có UX/retry xử lý conflict.
- ❌ Contention cao thì retry liên tục, tệ hơn pessimistic.
- ⚠️ Dùng `updated_at` làm version thì mọi đường ghi _khác_ (job nền, sync POS) chạm vào row đều làm client đang mở form bị stale — phải chấp nhận hoặc dùng version column riêng.

### 2.3. Distributed lock (Redis `SET NX EX`) — họ hàng của pessimistic, nhưng đứng riêng

Cùng triết lý "chặn trước" (ai không giành được lock thì không được làm), nhưng nằm ở **tầng ứng dụng**, dùng khi cần khóa **xuyên qua nhiều instance/process** mà không muốn mở DB transaction:

```
SET lock:pacing:venue-9:dinner:2026-08-17 <token> NX EX 30
```

- `NX` = chỉ set nếu key chưa tồn tại → ai set được là người giữ lock.
- `EX` = TTL, bảo hiểm khi process giữ lock chết giữa chừng.

Lý do tách riêng, không xếp chung nhóm với DB lock — khác bản chất ở ba điểm:

|            | DB pessimistic lock                         | Redis SETNX                                |
| ---------- | ------------------------------------------- | ------------------------------------------ |
| Ai đến sau | **chờ** rồi vẫn được xử lý                  | **fail ngay** (hoặc tự retry)              |
| Vòng đời   | gắn transaction, tự nhả khi commit/rollback | theo TTL, hết giờ là mất dù việc chưa xong |
| Bảo đảm    | tuyệt đối (correctness)                     | best-effort                                |

Nhược điểm cố hữu: TTL hết trước khi việc xong → hai bên cùng tưởng mình giữ lock. Redis lock **không có fencing token** nên không tuyệt đối an toàn như DB lock — chỉ dùng cho dedupe/pacing, không dùng làm hàng rào cuối cùng cho correctness tiền bạc; chốt chặn cuối vẫn phải là DB (lock hoặc unique constraint).

### 2.4. Constraint-based — exclusion constraint (`EXCLUDE USING gist`)

Chiến lược thứ ba, ít được dạy hơn nhưng mạnh nhất khi áp dụng được: **viết luật bất biến thành constraint ở tầng DB, để DB tự làm trọng tài**. Không cần app khóa gì, không cần version — app quên check cũng không thể ghi ra dữ liệu sai.

Dạng quen thuộc nhất là UNIQUE constraint (chống double-insert). Exclusion constraint là UNIQUE **tổng quát hóa**: UNIQUE chỉ so được "bằng nhau" (`=`), exclusion so được bất kỳ toán tử nào — quan trọng nhất là `&&` (hai khoảng giao nhau) trên range:

```sql
ALTER TABLE bookings
  ADD CONSTRAINT no_overlap
  EXCLUDE USING gist (
    table_id WITH =,      -- cùng bàn
    span     WITH &&      -- và khoảng thời gian chồng nhau
  ) WHERE (deleted_at IS NULL);
-- = "không bao giờ tồn tại 2 booking active cùng bàn chồng giờ nhau"
```

**GiST là gì:**

GiST (Generalized Search Tree) là một **loại index** của Postgres, thiết kế cho các kiểu dữ liệu phức tạp — khoảng (range), không gian/tọa độ (geometry, PostGIS), mảng, full-text (`tsvector`)… Khác biệt cốt lõi so với B-tree:

- **B-tree** chỉ hiểu thứ tự tuyến tính một chiều → trả lời nhanh `=`, `<`, `>`, `BETWEEN` trên số/chuỗi/ngày.
- **GiST** hiểu quan hệ nhiều chiều → trả lời nhanh các phép **giao nhau** (`&&`), **chứa nhau** (`@>`, `<@`), **khoảng cách/gần nhất** (`<->`). Câu hỏi "khoảng nào giao khoảng này" hay "điểm nào gần điểm này nhất" B-tree không trả lời được, GiST xử lý hiệu quả.

(Liên quan: GIN là loại index thường dùng hơn cho JSONB/mảng khi chỉ cần tra "có chứa phần tử không"; GiST phù hợp hơn cho range và khoảng cách.)

GiST tự nó **không phải cơ chế lock** — nó xuất hiện trong chương này chỉ vì exclusion constraint cần một index hiểu được toán tử `&&` để enforce luật "không chồng nhau", mà đó là việc B-tree không làm được.

**Hành vi khi đua:** hai transaction cùng insert span chồng nhau → transaction sau block chờ transaction trước commit (Postgres tự lock nội bộ khi check), rồi nhận lỗi `23P01 exclusion_violation`. Từ góc nhìn app nó giống optimistic — không giữ lock trước, bên thua nhận lỗi lúc ghi — nhưng không có kẽ hở nào để lọt.

**Vị trí trong bức tranh:** không thay thế hai chiến lược kia mà làm **lưới an toàn cuối cùng** phía sau chúng. Lock/version có thể bị bug ở tầng app; constraint thì không đường nào vòng qua được. Đổi lại, nó chỉ diễn đạt được các luật dạng "không tồn tại 2 row thỏa X" — logic phức tạp hơn vẫn phải dùng lock.

### 2.5. Chọn loại nào — quy tắc kinh nghiệm

| Tiêu chí          | Pessimistic                                 | Optimistic                               |
| ----------------- | ------------------------------------------- | ---------------------------------------- |
| Tần suất xung đột | Cao / chắc chắn (giờ cao điểm)              | Hiếm                                     |
| Cửa sổ xung đột   | Ngắn — vài chục ms trong 1 request          | Dài — theo phiên thao tác của người dùng |
| Người thua nên    | Chờ rồi vẫn được xử lý                      | Bị báo conflict, tự reload               |
| Hệ quả nếu sai    | Không chấp nhận được (tiền, double-booking) | Chấp nhận được (sửa lại lần nữa)         |
| Ví dụ điển hình   | Trừ kho, giành slot, đếm quota              | Sửa hồ sơ, CMS, kanban                   |

Câu hỏi quyết định: **"lock cần sống bao lâu?"** Sống trong 1 request → pessimistic được. Sống qua thời gian người dùng suy nghĩ → bắt buộc optimistic.

Hai loại **không loại trừ nhau** — một hệ thống lành mạnh dùng cả hai cho hai lớp bài toán khác nhau (nollie-api là ví dụ, xem phần 3).

---

## 3. nollie-api đang dùng gì

Bốn nhóm: **pessimistic trong DB** (3.1 — ba tầng, áp đảo về số lượng), **distributed lock Redis** (3.2 — tầng ứng dụng, best-effort), **optimistic** (3.3 — một cơ chế duy nhất) và **exclusion constraint** (3.4 — lưới an toàn cuối); 3.5 tóm tắt cách phân công.

### 3.1. Pessimistic trong DB — ba tầng

#### a) Row lock Sequelize — `lock: transaction.LOCK.UPDATE` (~22 chỗ)

Pessimistic chuẩn, luôn đi kèm transaction:

```typescript
const usage = await this.venueUsageRepository.findOne({
  where: { venueId },
  transaction,
  lock: transaction.LOCK.UPDATE,
});
```

Điểm dùng chính:

- `src/modules/venue-usages/venue-usages.service.ts` — đếm/trừ quota (read-modify-write kinh điển, 6 chỗ)
- `src/modules/reservation-hold/reservation-hold.service.ts` — khóa row hold khi convert hold → booking, hai lần convert song song thì một bên thắng
- `src/modules/venues/venues.service.ts`, `brand-identities`, scheduler automation…

#### b) Raw `SELECT ... FOR UPDATE`

Khi cần khóa một _tập_ row hoặc khóa trong câu SQL phức tạp:

- `src/modules/reservation/helpers/reservation-table-assignment.helper.ts` — khóa các row `reservation_restaurant_table` khi gán bàn, serialize việc chọn bàn (lưới cuối chống double-booking là exclusion constraint ở 3.4)
- `src/modules/reservation-promotion/reservation-promotion.service.ts` — khóa row cha để serialize các PUT song song trên cùng promotion

#### c) Advisory lock — `pg_advisory_xact_lock(hashtext(key))` (4 chỗ)

Serialize theo key logic khi không có row để khóa (hoặc row chưa tồn tại):

- `src/modules/reservation/helpers/reservation-pos-party-lock.helper.ts` — mọi xử lý cho cùng một POS party đi tuần tự
- `src/modules/customers/customer.service.ts` (2 chỗ), `campaign-v2.service.ts` (1 chỗ)

### 3.2. Distributed lock Redis — `redis.set(key, value, 'EX', ttl, 'NX')`

`src/services/redis/redis.service.ts` — dùng cho pacing lock của booking (key theo venue+service+date, chung giữa widget confirm và agent create), dedupe queue job, campaign send. Đúng vai trò: chống trùng ở tầng ứng dụng, **không phải** hàng rào correctness cuối cùng (hàng rào cuối vẫn là DB lock ở 3.1).

### 3.3. Optimistic — guard `clientUpdatedAt` (luồng staff sửa reservation, ~5 call site)

Không dùng version column; tự chế theo `updated_at`. FE gửi kèm `clientUpdatedAt` là `updatedAt` lúc đọc; service so khớp trước khi ghi:

```typescript
// src/modules/reservation/reservation.service.ts — assertNotStale()
if (new Date(clientUpdatedAt).getTime() !== currentUpdatedAt.getTime()) {
  // 409 — reservation đã bị người khác sửa, FE reload
}
```

Áp dụng cho: update reservation, cancel, transition status, transition arrival status, swap slot (swap check `updatedAt` của **cả hai** reservation). Lý do chọn optimistic ở đây: staff mở form/kéo kanban trong nhiều giây đến nhiều phút — không thể giữ row lock suốt thời gian đó, và xung đột giữa hai staff là hiếm.

Lưu ý codebase: có chỗ cần bump dữ liệu mà **không** được chạm `updated_at` (vì sẽ phá guard này) — xem comment ở `src/database/entities/reservations.model.ts` quanh dòng 283.

### 3.4. Exclusion constraint (GiST) — lưới an toàn cuối chống double-booking

Hai bảng span dùng cùng pattern, mirror nhau:

- `reservation_hold_physical_span` (hold) — migration `20260514120000`
- `reservation_table_physical_span` (booking đã confirm) — migration `20260526100000`

```sql
ALTER TABLE reservation_table_physical_span
  ADD CONSTRAINT excl_rs_table_phys_span_no_overlap
  EXCLUDE USING gist (
    venue_id            WITH =,
    restaurant_table_id WITH =,
    span                WITH &&
  ) WHERE (deleted_at IS NULL);
```

= không bao giờ tồn tại hai row active có cùng bàn mà `span` (khoảng `[start, end)` half-open, UTC) chồng nhau. Cancel/complete thì soft-delete row span để giữ audit history, và `WHERE (deleted_at IS NULL)` loại row đã xóa khỏi luật.

Đây là tầng **bất khả xâm phạm** — kể cả khi pacing lock lẫn `FOR UPDATE` ở trên có bug, DB vẫn từ chối ghi double-booking.

Phân biệt: các index `GIST (venue_id, span)` khác trong repo (vd. migration timeline-performance-indexes) chỉ là index thuần cho query `span && range` chạy nhanh — không liên quan locking.

### 3.5. Bức tranh phân công

Chuỗi phòng thủ chống double-booking, từ ngoài vào trong:

```
Redis pacing lock      →  giảm va chạm, best-effort
FOR UPDATE (gán bàn)   →  serialize việc chọn bàn
GiST exclusion         →  luật bất biến, không vòng qua được
```

Phân công theo bài toán:

```
Xung đột tức thời, cửa sổ ms, hệ quả nặng   →  pessimistic (DB lock)
  giành bàn, convert hold, quota, POS party

Chống trùng xuyên instance, best-effort      →  Redis SETNX + TTL
  pacing, dedupe job, campaign send

Xung đột hiếm, cửa sổ dài theo phiên người   →  optimistic (clientUpdatedAt)
  staff sửa/cancel/kéo reservation

Luật bất biến diễn đạt được bằng constraint  →  exclusion constraint (GiST)
  không double-booking một bàn vật lý
```

---

## 4. Bẫy thực tế (đã gặp hoặc dễ gặp trong repo này)

1. **Kiểm tra điều kiện trước khi lock = vô nghĩa.** Điều kiện phải được kiểm tra lại _sau_ khi `FOR UPDATE` trả về, vì lúc chờ lock dữ liệu có thể đã đổi. Lock → re-read (chính câu SELECT lock đó) → check → write.
2. **Quên `transaction` khi truyền `lock`.** Sequelize lock chỉ có nghĩa trong transaction; thiếu transaction thì hoặc lỗi hoặc âm thầm không khóa gì. Codebase có pattern `...(transaction && { lock: transaction.LOCK.UPDATE, transaction })` — lock và transaction luôn đi cặp.
3. **Global `query.raw = true` của repo này:** row lấy về là plain object, không gọi `.update()`/`.destroy()` được. Chỗ nào lock-rồi-ghi trên instance phải thêm `raw: false` (xem convention chung của codebase).
4. **Giữ DB lock qua external call.** Không gọi Stripe/POS/SendGrid khi đang giữ `FOR UPDATE` — thời gian giữ lock phải tính bằng ms. Việc chậm thì tách ra ngoài transaction hoặc đẩy vào queue.
5. **Deadlock do thứ tự khóa.** Khóa nhiều row (vd. nhiều bàn, hoặc swap 2 reservation) thì luôn khóa theo thứ tự id tăng dần.
6. **Redis TTL không phải giấy phép vĩnh viễn.** Việc chạy lâu hơn TTL thì lock coi như mất — đừng đặt logic "chỉ một người được làm" quan trọng chỉ dựa vào Redis lock; DB (unique constraint / DB lock) mới là chốt chặn cuối. Pattern trong repo: send-log unique claim làm chốt, Redis chỉ giảm va chạm.
7. **`updated_at` làm version = mọi writer đều đụng guard.** Job nền hoặc POS sync sửa reservation sẽ làm staff đang mở form bị 409 dù không ai "thật sự" tranh chấp. Đây là trade-off đã chấp nhận; nếu về sau thành vấn đề thì chuyển sang version column riêng.

---

## 5. Checklist khi viết code mới có shared write

- [ ] Có race không? (2 request song song cùng đọc-rồi-ghi một thứ)
- [ ] Cửa sổ xung đột ngắn (trong request) → pessimistic; dài (theo phiên user) → optimistic
- [ ] Pessimistic: có transaction chưa? check điều kiện _sau_ lock chưa? thứ tự khóa cố định chưa? có external call trong lúc giữ lock không?
- [ ] Optimistic: client có gửi version/`updatedAt` không? conflict trả 409 với code rõ ràng chưa? FE có đường reload không?
- [ ] Cần khóa mà row chưa tồn tại / khóa theo khái niệm logic → advisory lock
- [ ] Chỉ cần chống trùng best-effort xuyên instance → Redis SETNX, và vẫn phải có chốt chặn DB phía sau
- [ ] Luật diễn đạt được dạng "không tồn tại 2 row thỏa X" → thêm exclusion/unique constraint làm lưới cuối, kể cả khi đã có lock (bắt lỗi `23P01`/`23505` và trả conflict có code rõ ràng)
