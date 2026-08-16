# Concurrency Control — Xử lý Race Condition & Đảm bảo Toàn vẹn Dữ liệu

Tài liệu lý thuyết để học và áp dụng cho nhiều dự án. Ví dụ minh hoạ lấy từ bài toán booking (đặt bàn nhà hàng), case study cuối bài là nollie-api, nhưng toàn bộ nguyên tắc là generic.

---

## 1. Nguyên lý cốt lõi

> **Mọi race condition đều là một cửa sổ read-then-write (TOCTOU — Time-Of-Check To Time-Of-Use), và mọi giải pháp chỉ làm một trong hai việc: bắt xếp hàng (serialize) hoặc cho chạy song song rồi xử kẻ thua lúc commit (detect-at-commit).**

Ví dụ chuẩn: 2 request cùng đặt bàn cuối cùng.

```
Request A: đọc bàn 5 → trống ✅        Request B: đọc bàn 5 → trống ✅
Request A: INSERT booking bàn 5        Request B: INSERT booking bàn 5
→ double booking. Cả hai đều "check rồi mới ghi", nhưng check của B
  xảy ra TRƯỚC KHI write của A commit → check vô nghĩa.
```

Hai triết lý xử lý, mọi công cụ đều rơi vào một trong hai:

| Triết lý | Ý tưởng | Khi nào thắng |
|---|---|---|
| **Pessimistic** (bi quan) | Chặn trước: ai đến sau phải chờ hoặc fail ngay | Conflict xảy ra **thường xuyên**, retry đắt |
| **Optimistic** (lạc quan) | Cho chạy song song, phát hiện xung đột lúc commit, kẻ thua rollback + retry/báo lỗi | Conflict **hiếm**, throughput quan trọng |

**Nguyên tắc vàng:** đặt *correctness* ở tầng thấp nhất có thể (database), các tầng trên (Redis, app) chỉ phục vụ UX và performance. Vì DB là nơi duy nhất mọi con đường ghi dữ liệu bắt buộc phải đi qua — app có thể có 10 instance, Redis có thể chết, nhưng câu INSERT cuối cùng luôn chạm DB.

---

## 2. Tổng quan các cơ chế kiểm soát đồng thời

| # | Công cụ | Tầng | Triết lý | Dùng khi |
|---|---|---|---|---|
| 1 | `SELECT … FOR UPDATE` (row lock) | DB | Pessimistic | Đã **có row tồn tại** để khoá, cần check-rồi-ghi trên row đó |
| 2 | Version column (optimistic lock) | DB + app | Optimistic | Update row có sẵn, conflict hiếm, chấp nhận retry |
| 3 | Advisory lock (`pg_advisory_xact_lock`) | DB | Pessimistic | Cần serialize theo **khái niệm logic** chưa có row nào đại diện |
| 4 | Constraint (`UNIQUE`, `EXCLUDE USING gist`) | DB | Optimistic (DB enforce) | Invariant biểu diễn được bằng constraint → **luôn** làm lưới an toàn cuối |
| 5 | Atomic single-statement | DB | — | Gộp check + write vào 1 câu SQL → không còn cửa sổ TOCTOU |
| 6 | Distributed lock (Redis `SET NX` / Redlock) | Redis | Pessimistic | Serialize tài nguyên **ngoài DB**, hoặc nhiều service không chung DB |
| 7 | Hold pattern (row giữ chỗ có TTL) | DB | Pessimistic (dạng dữ liệu) | User cần "giữ chỗ" **qua nhiều request** (điền form, thanh toán) |
| 8 | Idempotency key | DB + app | — | Chống race dạng **duplicate**: retry, double-click, webhook gửi lại |

---

## 3. Phân tích chi tiết từng cơ chế

### 3.1. Pessimistic Locking — `SELECT … FOR UPDATE`

**Cơ chế:** khoá row ở mức DB trong transaction. Transaction thứ hai đụng cùng row sẽ **chờ** đến khi transaction đầu commit/rollback, sau đó đọc được dữ liệu mới nhất → check conflict chạy lại trên dữ liệu đúng.

```sql
BEGIN;
SELECT * FROM tables WHERE id = 5 FOR UPDATE;   -- B phải chờ ở đây
-- check overlap ... không có ai → an toàn, vì không ai chen được vào nữa
INSERT INTO bookings (table_id, ...) VALUES (5, ...);
COMMIT;                                          -- B tỉnh dậy, thấy booking của A → fail check
```

**Rủi ro quan trọng nhất — deadlock:** nếu transaction A khoá row 1 rồi chờ row 2, còn B khoá row 2 rồi chờ row 1 → deadlock. **Giải pháp: luôn khoá theo một thứ tự toàn cục cố định** (ví dụ: sort theo `id` tăng dần trước khi khoá).

```sql
-- Khoá nhiều bàn cho combo: BẮT BUỘC ORDER BY id
SELECT id FROM tables WHERE id IN (5, 3, 8) ORDER BY id FOR UPDATE;
```

**Ưu:** đơn giản, đúng tuyệt đối, dữ liệu đọc sau lock luôn mới nhất.
**Nhược:** giảm throughput (xếp hàng), chỉ dùng được khi **row đã tồn tại**, giữ lock lâu thì cả hàng chờ theo.

### 3.2. Optimistic Locking — version column

**Cơ chế:** thêm cột `version`. Update kèm điều kiện version cũ; nếu `affected rows = 0` nghĩa là ai đó đã sửa trước → rollback, báo lỗi hoặc retry.

```sql
UPDATE bookings
SET table_id = 5, version = version + 1
WHERE id = 42 AND version = 7;
-- affected = 0 → conflict, người khác đã update trước
```

**Ưu:** không giữ lock, throughput cao, không deadlock.
**Nhược:** chỉ bảo vệ **update row có sẵn** (không chống được 2 INSERT song song); phải viết logic retry; conflict nhiều thì retry storm còn tệ hơn pessimistic.

**Lưu ý:** nếu invariant của bạn biểu diễn được bằng constraint (mục 3.4) thì constraint cho cùng semantics "kẻ thua fail lúc commit" mà **DB tự enforce**, không cần app tự quản version — thường là lựa chọn tốt hơn.

### 3.3. Advisory Lock — khoá theo khái niệm logic, không theo row

**Vấn đề mà FOR UPDATE không giải được:** check dạng aggregate trước khi INSERT row **mới**. Ví dụ pacing: "tổng khách trong khung giờ này ≤ 30 thì mới cho đặt". Không có row nào đại diện cho "khung giờ này" để mà FOR UPDATE — 2 transaction cùng `SUM()` ra 28, cùng INSERT party 4 → 36, vỡ cap.

**Cơ chế:** Postgres cho phép khoá theo **một con số tuỳ ý** — bạn tự đặt tên cho khái niệm cần serialize:

```sql
-- Khoá theo key logic "venue 12, service 3, ngày 2026-08-16"
SELECT pg_advisory_xact_lock(hashtext('pacing:12:3:2026-08-16'));
-- _xact = tự release khi transaction commit/rollback, không thể quên unlock
SELECT SUM(party_size) ...;   -- giờ mới đọc, đã độc quyền
INSERT INTO bookings ...;
```

**Best practices:**
- Luôn dùng bản `_xact` (transaction-scoped) — tự release, không leak lock khi app crash.
- Đặt `lock_timeout` để **fail fast** thay vì treo cả hàng đợi sau một transaction chậm:

```sql
SET LOCAL lock_timeout = '2s';
-- timeout → bắt error code 55P03 → trả 503 "thử lại sau" cho client
```

- Khoá **đúng scope** của invariant: cap tính theo (venue, service, ngày) thì key phải là (venue, service, ngày) — key hẹp hơn (per-slot) vẫn cho 2 slot khác nhau cùng vượt cap của service.
- Acquire lock **càng muộn càng tốt**: làm hết việc chậm (gọi API ngoài, xử lý nặng) trước, rồi mới lock → cửa sổ serialize nhỏ nhất.

### 3.4. Database Constraint — lưới an toàn cuối cùng

**Cơ chế:** mô tả invariant cho DB, DB tự chặn ở mức atomic — 2 row vi phạm **không bao giờ** cùng commit được, bất kể app bug gì phía trên.

`UNIQUE` cho invariant dạng "chỉ một":

```sql
-- Một bàn chỉ có 1 booking active tại một thời điểm chính xác
CREATE UNIQUE INDEX ON bookings (table_id, start_time) WHERE status = 'active';
```

`EXCLUDE USING gist` (Postgres) — bản tổng quát của UNIQUE, cho invariant dạng "không chồng lấn **khoảng**":

```sql
-- Không cho 2 booking cùng bàn có khoảng thời gian giao nhau
ALTER TABLE booking_spans ADD CONSTRAINT no_overlap
EXCLUDE USING gist (
  table_id WITH =,
  span     WITH &&        -- span là tstzrange [start, end+buffer)
);
-- Kẻ thua nhận error 23P01 (exclusion_violation) lúc commit → catch → báo "hết chỗ"
```

**Đây chính là optimistic locking do DB enforce:** cho chạy song song, kẻ thua fail lúc commit — nhưng không cần version column, không cần app làm gì ngoài catch error.

**Nguyên tắc:** kể cả khi đã có lock ở các tầng trên, **nếu invariant biểu diễn được bằng constraint thì luôn thêm constraint**. Lock ở tầng trên có thể bị quên ở một code path mới; constraint thì không đường ghi nào né được.

**Hạn chế:** constraint phải immutable — không tham chiếu được `now()`. Hệ quả: row "đã hết hạn" vẫn chiếm chỗ trong constraint đến khi bị xoá → cần background job dọn (xem 3.7).

### 3.5. Atomic Single-Statement — loại bỏ cửa sổ TOCTOU

Nhiều race biến mất khi gộp check + write vào **một câu SQL** — DB thực thi từng câu atomic:

```sql
-- Trừ kho: check và trừ trong 1 câu, không cần lock
UPDATE inventory SET stock = stock - 1
WHERE product_id = 7 AND stock > 0;
-- affected = 0 → hết hàng

-- Claim job: chỉ 1 worker thắng
UPDATE jobs SET claimed_by = :worker
WHERE id = :id AND claimed_by IS NULL;

-- Insert-nếu-chưa-có + tăng counter atomic
INSERT INTO counters (key, value) VALUES (:k, 1)
ON CONFLICT (key) DO UPDATE SET value = counters.value + 1
RETURNING value;
```

Luôn tự hỏi "có gộp được thành 1 câu không?" **trước khi** với tới lock — rẻ nhất, nhanh nhất, không deadlock.

### 3.6. Distributed Lock — Redis `SET NX` / Redlock

**Cơ chế:** `SET key value NX EX ttl` — chỉ set được nếu key chưa tồn tại, atomic. Ai set được là người giữ lock; TTL để lock tự chết nếu holder crash.

```
SET lock:table:5:slot:19h "token-cua-toi" NX EX 10
→ OK      = lấy được lock
→ nil     = ai đó đang giữ
```

Release **bắt buộc** phải compare-and-delete bằng Lua script (atomic) — nếu chỉ `DEL` thẳng, bạn có thể xoá nhầm lock của người khác khi TTL của mình đã hết giữa chừng:

```lua
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else
  return 0
end
```

**Redlock — cơ chế bên trong:** một node Redis duy nhất là single point of failure (node chết / failover sang replica chưa kịp sync → lock "bốc hơi", 2 client cùng giữ). Redlock giải bằng cách lấy lock trên **đa số** của N node Redis **độc lập** (không replicate nhau, thường N=5):

```
1. Ghi lại thời điểm bắt đầu T1
2. Tuần tự SET NX EX lên cả 5 node (mỗi lần có timeout ngắn ~50ms,
   node chết thì bỏ qua ngay, không chờ)
3. Lấy được ≥ 3/5 node VÀ tổng thời gian đi xin < TTL
   → lock hợp lệ, thời gian dùng thực = TTL − (now − T1) − clock_drift
4. Không đủ đa số → DEL trên TẤT CẢ các node (kể cả node đã OK) → retry sau delay ngẫu nhiên
5. Release = Lua compare-and-delete trên tất cả các node
```

Chịu được (N−1)/2 node chết mà lock vẫn đúng. Nhưng **vẫn không kín kẽ tuyệt đối** — đây là tranh luận kinh điển Kleppmann vs antirez (2016), đáng nhắc trong phỏng vấn:

- **Kleppmann:** Redlock phụ thuộc giả định về thời gian (clock các node trôi ít, GC pause ngắn). Một GC pause dài giữa lúc "check lock còn hạn" và lúc "ghi DB" → TTL đã hết, client khác đã vào, mà client cũ không hề biết → vẫn ghi đè. Muốn đúng tuyệt đối phải có **fencing token**: mỗi lần cấp lock kèm số tăng đơn điệu, storage phía dưới từ chối write mang token cũ hơn token đã thấy — tức là chốt chặn cuối vẫn phải nằm ở storage, không phải ở lock.
- **antirez (tác giả Redis):** clock drift thực tế đủ nhỏ nếu vận hành tử tế, và fencing đúng nghĩa thì bản thân storage đã phải có cơ chế ordering — lúc đó cần gì lock nữa.
- **Bài học rút ra (dùng được cho mọi dự án):** lời phê của Kleppmann đúng về bản chất — *lock phân tán dựa trên TTL chỉ nên làm tầng efficiency (đỡ việc thừa), còn correctness phải do storage cuối cùng enforce* — trong Postgres, "fencing token" chính là row lock / constraint / version check. Đây là lý do lý thuyết đứng sau nguyên tắc vàng ở §1.

**Điểm yếu bản chất:**
- Lock chỉ là **advisory**: mất khi Redis restart/failover.
- **TTL hết hạn giữa chừng** khi transaction còn đang chạy (GC pause, query chậm) → lock âm thầm mở cho người khác vào → race quay lại, mà không ai biết.
- Vì vậy: **không bao giờ dùng Redis lock làm chốt chặn correctness duy nhất.** Đúng vai của nó:
  - Chống **cache stampede** (100 request cùng cache-miss → chỉ 1 thằng compute, số còn lại chờ/dùng cache cũ).
  - Serialize tài nguyên ngoài DB (gọi API bên thứ ba có rate limit, cron chỉ chạy 1 instance).
  - Giảm tải cho DB lock (chặn bớt từ xa), nhưng phía sau vẫn phải có DB layer đỡ.

### 3.7. Hold Pattern — giữ tài nguyên qua nhiều request

**Vấn đề:** DB lock và Redis lock đều sống trong **một** transaction/khoảnh khắc. Nhưng flow đặt bàn thật là: chọn slot → điền form 5 phút → thanh toán → confirm. Không thể giữ transaction mở 5 phút.

**Cơ chế:** biến "lock" thành **dữ liệu** — một row `hold` có TTL:

```
POST /holds        → INSERT hold row (expires_at = now + 10 phút)
                     + các row "span" chiếm chỗ trong exclusion constraint (3.4)
                     → từ giây này, allocator coi bàn đó là bận
POST /bookings     → SELECT hold FOR UPDATE (chống double-convert)
                     → validate chưa hết hạn → tạo booking → xoá span → đánh dấu converted
Background job     → 30s/lần, dọn hold hết hạn (vì constraint không tự biết now())
```

So với "Redis lock giữ chỗ khi thanh toán": hold row **bền** (sống qua restart), **audit được** (ai giữ, lúc nào, convert hay bỏ), và được chính exclusion constraint bảo vệ chống 2 hold chồng nhau. Đây là câu trả lời đúng cho mọi bài toán "giữ tài nguyên trong lúc user suy nghĩ": ghế máy bay, vé concert, giỏ hàng flash-sale.

### 3.8. Idempotency Key — chống race dạng duplicate

Race không chỉ là 2 người tranh 1 tài nguyên — còn là **1 người gửi 2 lần** (double-click, mobile retry, webhook gửi lại). Giải pháp: client gửi kèm key duy nhất, server nhớ key đã xử lý:

```sql
-- UNIQUE constraint làm luôn việc check
CREATE UNIQUE INDEX ON requests (idempotency_key);
-- Request 2 với cùng key → unique violation → trả lại response cũ, không xử lý lại
```

Webhook handler chuẩn: check idempotency → ack nhanh → đẩy việc nặng vào queue.

---

## 4. Tiêu chí lựa chọn cơ chế

Đi theo thứ tự câu hỏi:

```
1. Gộp check + write vào 1 câu SQL được không?
   → Được: atomic single-statement (3.5). Xong, khỏi lock.

2. Invariant biểu diễn được bằng UNIQUE / EXCLUDE constraint không?
   → Được: LUÔN thêm constraint làm backstop (3.4),
     rồi mới cân nhắc có cần lock phía trên để UX đẹp hơn không
     (lock cho lỗi sớm + message rõ; constraint cho đúng tuyệt đối).

3. Cần serialize check-rồi-ghi. Row để khoá đã tồn tại chưa?
   → Có row      : SELECT ... FOR UPDATE (3.1) — nhớ lock ordering.
   → Chưa có row : advisory lock theo key logic (3.3) — nhớ lock_timeout.

4. Là update row có sẵn, conflict hiếm, muốn throughput?
   → Optimistic version column (3.2) + retry.

5. Tài nguyên NGOÀI DB, hoặc nhiều service không chung DB, hoặc chống cache stampede?
   → Redis SET NX (3.6) — và nhắc lại: không làm chốt correctness duy nhất.

6. User cần giữ tài nguyên QUA NHIỀU REQUEST (form, payment)?
   → Hold pattern (3.7) — lock không sống qua request, dữ liệu thì có.

7. Race dạng duplicate (retry, double-click, webhook)?
   → Idempotency key (3.8).
```

Và nguyên tắc xuyên suốt: **các tầng chồng lên nhau, không thay thế nhau.** Hệ thống tốt thường có: advisory/row lock để serialize + fail sớm với message đẹp, constraint để không gì lọt qua được, Redis chỉ đứng ở tầng performance.

---

## 5. Các lỗi thiết kế thường gặp

| Bẫy | Triệu chứng | Phòng |
|---|---|---|
| Deadlock | 2 transaction treo rồi 1 bị DB kill (`40P01`) | Khoá nhiều row theo **thứ tự toàn cục cố định** (sort id) |
| Lock window quá to | Cả hệ thống xếp hàng sau 1 request chậm | Làm việc nặng (API ngoài, tính toán) **trước** khi lock; lock muộn nhất có thể |
| Chờ lock vô hạn | Request treo, connection pool cạn | `SET LOCAL lock_timeout` → fail fast → client retry |
| Redis TTL hết giữa chừng | Double booking "không thể xảy ra" xuất hiện lác đác | Không dùng Redis làm chốt correctness; DB constraint đỡ sau lưng |
| DEL nhầm lock người khác | Lock "mất tác dụng" ngẫu nhiên | Release bằng Lua compare-and-delete với token riêng |
| In-memory mutex khi chạy nhiều instance | Chạy dev thì đúng, lên prod (N pod) thì race | Mutex trong process chỉ đúng với 1 instance — phải xuống DB/Redis |
| Optimistic không retry | User thấy lỗi conflict cụt lủn | Version-mismatch phải có chiến lược: retry tự động hoặc báo lỗi có hướng dẫn |
| Check ở app, quên ở DB | Code path mới (API mới, script backfill) né mất check | Constraint ở DB — đường nào cũng phải chui qua |
| Constraint chứa now() | `functions in index expression must be immutable` | Row hết hạn vẫn chiếm chỗ → background job dọn định kỳ |

---

## 6. Case study: booking engine của nollie-api

Minh hoạ nguyên tắc "correctness ở DB, layer chồng nhau" trong code thật:

| Tầng | Công cụ (mục) | Áp dụng |
|---|---|---|
| 1 | Advisory lock (3.3) | `pg_advisory_xact_lock` theo key `pacing-service-day:{venue}:{service}:{date}` serialize check pacing cho booking không gắn bàn; `lock_timeout 2s` → 503 retryable |
| 2 | Row lock (3.1) | `FOR UPDATE` trên row bàn vật lý trước check overlap; combo nhiều bàn khoá theo id tăng dần chống deadlock |
| 3 | Row lock (3.1) | `FOR UPDATE` trên hold row lúc convert — 2 request cùng holdToken thì kẻ thua nhận 410 |
| 4 | Constraint (3.4) | `EXCLUDE USING gist (venue, table, span &&)` trên cả hold spans lẫn booking spans — backstop cuối, lỗi `23P01` |
| 5 | Hold pattern (3.7) | Widget giữ bàn 10 phút bằng hold row + span, background job dọn hold hết hạn mỗi 30s |
| — | Redis (3.6) | Chỉ 1 chỗ: `SET NX` + Lua release làm compute lock cho allocator cache (chống stampede) — thuần performance |

Hai lựa chọn "không dùng" đáng học:
- **Không Redlock cho booking:** vai trò giữ chỗ đã có hold row đảm nhận, bền và audit được hơn.
- **Không version column:** exclusion constraint cho cùng semantics kẻ-thua-fail-lúc-commit, DB tự enforce.

---

## 7. Mẫu trả lời phỏng vấn — scenario "N client tranh 1 bàn cuối"

> *Câu hỏi: 2 khách hàng và 1 nhân viên nội bộ cùng lúc đặt cùng một bàn cuối, cùng khung giờ. Xử lý race condition ở tầng database (PostgreSQL/Redis) thế nào?*

Script trả lời ~2 phút, đi từ khung → chi tiết → phản biện:

**Mở — đặt khung (15 giây):**

> "Bản chất đây là bài toán read-then-write: cả 3 request đều đọc thấy bàn trống rồi mới ghi, và check của người này chạy trước khi write của người kia commit. Em xử lý bằng nhiều tầng chồng lên nhau, với nguyên tắc: correctness phải nằm ở Postgres — nơi mọi con đường ghi bắt buộc đi qua — còn Redis chỉ đứng ở tầng performance."

**Thân — đi qua từng tầng (60–90 giây):**

> "**Tầng 1 — Pessimistic lock:** trong transaction ghi booking, em `SELECT … FOR UPDATE` trên row bàn vật lý trước khi chạy check overlap. Ba transaction đụng cùng row sẽ xếp hàng; người thứ hai tỉnh dậy đọc được booking vừa commit của người thắng → fail check sạch sẽ. Nếu booking gộp nhiều bàn thì khoá theo thứ tự id tăng dần để tránh deadlock.
>
> "**Tầng 2 — trường hợp FOR UPDATE không với tới:** những check dạng aggregate — ví dụ cap tổng khách của khung giờ — không có row nào để khoá. Em dùng `pg_advisory_xact_lock` theo key logic `(venue, service, ngày)`, kèm `lock_timeout` 2 giây để fail fast trả 503 retryable thay vì treo cả hàng đợi.
>
> "**Tầng 3 — backstop bằng constraint:** mỗi booking ghi một span `tstzrange` cho từng bàn, có `EXCLUDE USING gist (table_id WITH =, span WITH &&)`. Kể cả app bug hay một code path mới quên lock, hai span chồng nhau không bao giờ cùng commit được — kẻ thua nhận `23P01` lúc commit. Đây thực chất là optimistic locking do DB tự enforce, nên em không cần version column riêng.
>
> "**Còn 'giữ bàn trong lúc khách điền form thanh toán':** em không dùng Redis lock cho việc này mà dùng hold pattern — một row hold có TTL 10 phút, span của nó chiếm chỗ trong chính exclusion constraint ở trên, background job dọn hold hết hạn. Convert hold thành booking thì `FOR UPDATE` trên hold row để chống double-submit."

**Chốt — phản biện Redlock (20 giây):**

> "Em cố tình **không** đặt Redlock làm chốt chặn: lock TTL-based mất khi Redis failover, và GC pause dài hơn TTL là lock âm thầm mở lại — đúng critique của Kleppmann, muốn kín phải có fencing token ở storage, mà trong Postgres thì row lock + constraint chính là fencing rồi. Redis em vẫn dùng, nhưng đúng vai: `SET NX` chống cache stampede cho availability cache — Redis chết thì chậm đi chứ không sai dữ liệu."

**Các câu hỏi đào sâu thường gặp:**

| Câu hỏi | Trả lời ngắn |
|---|---|
| "Sao không dùng optimistic version column?" | Chỉ bảo vệ update row có sẵn — đây là bài toán 2 INSERT song song; exclusion constraint cho cùng semantics mà DB tự enforce (§3.2, §3.4) |
| "Serializable isolation level có thay được không?" | Được về lý thuyết, nhưng phải retry mọi transaction bị `40001` và trả giá throughput toàn hệ thống; lock chọn lọc + constraint rẻ hơn và scope hẹp hơn |
| "Nhân viên nội bộ đi đường khác khách thì sao?" | Chính vì thế constraint phải ở DB — staff bypass hold nhưng không bypass được exclusion constraint; mọi entry point chung một chốt chặn cuối |
| "Nếu bàn không được gán lúc đặt (table-less)?" | Không có row để khoá → advisory lock theo key logic đúng scope của invariant (§3.3) |

---

## 8. Checklist khi review code

- [ ] Có đoạn nào read-rồi-write mà không có lock/constraint che chắn? (tìm pattern: `find` → `if` → `create/update`)
- [ ] Check aggregate (SUM/COUNT) trước INSERT → có advisory lock đúng scope chưa?
- [ ] Khoá nhiều row → có lock ordering chưa?
- [ ] Lock có `lock_timeout` / fail-fast chưa, hay sẽ treo cả hàng?
- [ ] Việc nặng có nằm ngoài vùng lock chưa?
- [ ] Invariant có constraint ở DB đỡ sau lưng chưa, hay chỉ app check?
- [ ] Redis lock có đang gánh correctness không? (nếu Redis chết/TTL hết thì có sai dữ liệu không?)
- [ ] Endpoint ghi có idempotency cho retry/double-click chưa?
- [ ] Row TTL (hold, lock dạng dữ liệu) có job dọn chưa?
