# Transactional Outbox & Change Data Capture (CDC)

Tài liệu học tập — lý thuyết về hai kỹ thuật đảm bảo data consistency khi một service phải vừa ghi DB vừa phát event, kèm ví dụ thực tế từ module `venue-registry` của nollie-api.

---

## 1. Bài toán gốc: Dual-Write Problem

Ý tưởng cốt lõi trong một câu: **không thể ghi vào hai hệ thống độc lập (DB + message broker / DB khác) một cách atomic, nên phải biến "hai lần ghi" thành "một lần ghi + một cơ chế lan truyền đáng tin cậy".**

Tình huống điển hình: khi tạo venue, ta cần (a) insert venue vào Postgres và (b) ghi routing entry `slug → region` vào DynamoDB Global Table để widget resolve đúng region. Hai hệ thống này không chia sẻ transaction, nên viết naive sẽ có 2 cách hỏng:

```typescript
// ❌ Cách 1: ghi DB xong rồi ghi Dynamo
await venueRepo.create(venue); // OK
await dynamo.put(slugEntry); // 💥 crash / network error
// → venue tồn tại nhưng widget không resolve được slug (mất event)

// ❌ Cách 2: ghi Dynamo trước rồi ghi DB
await dynamo.put(slugEntry); // OK
await venueRepo.create(venue); // 💥 transaction rollback
// → Dynamo có slug "ma" trỏ tới venue không tồn tại (event thừa)
```

Đặt `dynamo.put` **bên trong** transaction Postgres cũng không cứu được: nếu commit Postgres fail sau khi Dynamo đã ghi, không có cách nào "rollback" Dynamo. Đây gọi là **dual-write problem** — mọi thứ tự đều có cửa sổ lỗi.

Hai lời giải phổ biến: **Transactional Outbox** (application-level) và **Change Data Capture** (infrastructure-level). Cả hai cùng một nguyên tắc: _chỉ ghi vào một nơi (DB nguồn), rồi để một tiến trình riêng lan truyền thay đổi với đảm bảo at-least-once_.

---

## 2. Transactional Outbox Pattern

### 2.1 Lý thuyết

**Outbox là gì?** Tên pattern mượn từ "hộp thư đi" của email: thư chưa gửi được thì nằm lại Outbox, tiến trình nền gửi sau — có thể muộn nhưng không bao giờ mất. Ở đây "outbox" là một bảng trong chính DB nghiệp vụ, mỗi row là một event chờ gửi sang hệ thống ngoài.

**Ý tưởng:** đổi bài toán "ghi 2 nơi atomic" (bất khả thi) thành "ghi 1 nơi atomic + chuyển phát lại tin cậy". Thay vì ghi trực tiếp sang hệ thống thứ hai, ta ghi **ý định** (event) vào bảng `outbox` **trong cùng transaction** với dữ liệu nghiệp vụ. Vì cùng một DB, atomicity là tuyệt đối: hoặc cả venue lẫn event cùng tồn tại, hoặc cả hai cùng không. Phần gửi đi qua network giao cho tiến trình nền retry đến khi thành công — cái giá là hệ thống đích trễ hơn DB nguồn một khoảng (**eventual consistency**).

```
┌────────────────── 1 Postgres transaction ──────────────────┐
│  INSERT INTO venues (...)                                  │
│  INSERT INTO venue_registry_outbox (..., status=PENDING)   │
└──────────────────────── COMMIT ────────────────────────────┘
                             │
                             ▼
              Relay (poller hoặc CDC) đọc outbox
                             │
                             ▼
                    Message broker (SQS)
                             │
                             ▼
          Consumer idempotent → ghi hệ thống đích
```

Ba thành phần bắt buộc:

1. **Outbox table** — lưu event như một row, có cột `status` để theo dõi vòng đời.
2. **Message relay** — tiến trình tách rời, đọc row pending và publish lên broker. Hai biến thể: _polling publisher_ (cron quét bảng — đơn giản, độ trễ = chu kỳ poll) hoặc _transaction-log tailing_ (dùng CDC đọc WAL — độ trễ thấp, hạ tầng phức tạp hơn).
3. **Idempotent consumer** — vì delivery là **at-least-once** (relay có thể publish trùng, SQS có thể deliver trùng), consumer phải xử lý lại một event mà không gây tác dụng phụ.

Tính chất nhận được:

- **Không mất event** — event đã commit thì chắc chắn sẽ được xử lý (eventually).
- **Không có event "ma"** — transaction rollback thì event cũng biến mất theo.
- **Eventual consistency** — hệ thống đích trễ hơn DB nguồn một khoảng (chu kỳ poll + queue latency). Không phù hợp nếu cần read-after-write tức thời.

### 2.2 Bảng outbox gồm những gì

Một bảng outbox điển hình có 4 nhóm cột — lấy chính `venue_registry_outbox` của repo làm ví dụ:

| Nhóm             | Cột                                                               | Vai trò                                                                                                                                      |
| ---------------- | ----------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| Định danh        | `id`, `created_at`, `updated_at`                                  | Khóa chính (chính là "message id" gửi lên broker); `created_at` để relay xử lý theo thứ tự FIFO                                              |
| Nội dung event   | `event_type`, `venue_id`, `venue_slug`, `previous_slug`, `region` | Event là gì và đủ dữ liệu để consumer replay không cần join thêm — `event_type` cho ngữ nghĩa (upsert/rename...), các cột còn lại là payload |
| Vòng đời         | `status`                                                          | Event đang ở đâu trong pipeline (bên dưới)                                                                                                   |
| Vận hành / debug | `attempts`, `last_error`, `processed_at`                          | Đếm số lần retry, lỗi gần nhất, thời điểm hoàn thành — để giám sát và điều tra khi event kẹt/fail                                            |

Nguyên tắc thiết kế payload: **row phải tự chứa đủ dữ liệu để xử lý lại về sau** (self-contained). Ví dụ event `rename` lưu cả `previous_slug` — nếu chỉ lưu slug mới rồi đọc bảng `venues` lúc xử lý, slug cũ đã bị ghi đè mất, không biết phải xóa entry nào trong Dynamo.

**Các status và vòng đời:**

```
             relay claim          gửi SQS + consumer xử lý OK
  PENDING ──────────────▶ PUBLISHING ──────────────▶ DONE
     ▲                        │
     │   kẹt > 5 phút         │
     └────────────────────────┘        đích từ chối vĩnh viễn
                                       (vd slug bị venue khác giữ)
  PENDING/PUBLISHING ─────────────────────────────▶ FAILED
```

| Status       | Nghĩa                                                                   | Ai chuyển vào                                                                       |
| ------------ | ----------------------------------------------------------------------- | ----------------------------------------------------------------------------------- |
| `pending`    | Đã commit cùng transaction nghiệp vụ, chờ gửi                           | `enqueueEvent` (mặc định khi insert)                                                |
| `publishing` | Relay đã claim và đang/đã đẩy lên SQS                                   | Cron relay (update có điều kiện `status = pending` để 2 instance không claim trùng) |
| `done`       | Consumer đã áp dụng thành công lên đích                                 | `processOutboxEvent` — trạng thái cuối, deliver trùng gặp `done` là no-op           |
| `failed`     | Đích từ chối vĩnh viễn, retry cũng vô ích (vd slug đã thuộc venue khác) | `processOutboxEvent`, kèm `last_error` ghi lý do — cần người xử lý tay              |

Lưu ý phân biệt: lỗi **transient** (network, throttle) _không_ chuyển sang `failed` — chỉ tăng `attempts` và throw để SQS redeliver; `failed` dành riêng cho lỗi **vĩnh viễn** mà retry không thể cứu. Nhầm hai loại này là bug kinh điển: transient → failed thì mất event oan, permanent → retry mãi thì kẹt queue.

### 2.3 Ví dụ trong nollie-api: `venue-registry`

Bối cảnh: bảng DynamoDB `venue-registry` (partition key `venue_slug`) là bảng routing toàn cục cho booking widget — từ slug suy ra region (au/uk) + venue id. Nguồn sự thật là Postgres từng region; Dynamo là bản chiếu cần được đồng bộ tin cậy.

**Bước 1 — Ghi outbox atomic với nghiệp vụ.** Khi tạo venue, `enqueueEvent` nhận `transaction` của caller và insert row outbox trong đúng transaction đó:

```typescript
// src/modules/venues/venues.service.ts (~L430, trong createVenue)
await this.venueRegistryService.enqueueEvent(
  {
    venueId: venue.id,
    venueSlug,
    region: application.targetRegion,
    eventType: EVenueRegistryOutboxEventType.UPSERT,
  },
  transaction, // ← cùng transaction với INSERT venues
);
```

```typescript
// src/modules/venue-registry/venue-registry.service.ts
async enqueueEvent(payload, transaction?): Promise<void> {
  await this.outboxRepository.create(
    { ...payload, status: EVenueRegistryOutboxStatus.PENDING },
    { transaction },
  );
}
```

Model outbox (`src/database/entities/venue-registry-outbox.model.ts`) mang đủ dữ liệu để replay: `venueId`, `venueSlug`, `previousSlug` (cho RENAME), `region`, `eventType` (UPSERT / DISABLE / ENABLE / RENAME), `status`, `attempts`, `lastError`, `processedAt`.

**Bước 2 — Relay bằng polling publisher.** Cron 30 giây quét row `PENDING`, claim sang `PUBLISHING`, rồi đẩy lên SQS. Message chỉ chứa `outboxId` + `venueId` (reference ID, không nhúng payload — đúng queue rule của repo):

```typescript
// venue-registry.service.ts
@Cron(CronExpression.EVERY_30_SECONDS)
async relayPendingRows(): Promise<void> {
  const rows = await this.outboxRepository.find({
    where: { status: PENDING }, limit: 100, order: [['createdAt', 'ASC']],
  });
  for (const row of rows) {
    // claim có điều kiện status=PENDING → chống 2 instance relay trùng nhau
    await this.outboxRepository.updateWithoutReturning(
      { status: PUBLISHING },
      { where: { id: row.id, status: PENDING } },
    );
    await this.sqsService.sendMessage(VENUE_REGISTRY_UPSERT, { outboxId, venueId }, queueUrl);
  }
}
```

Kèm một cron "cứu hộ": row kẹt ở `PUBLISHING` quá 5 phút (relay chết giữa chừng sau claim, trước khi gửi SQS) được trả về `PENDING` để thử lại — `resetStuckRows()`. Đây là chi tiết dễ quên nhất khi tự cài outbox: **claim mà không có cơ chế thu hồi claim = event kẹt vĩnh viễn**.

**Bước 3 — Consumer idempotent.** SQS listener (`src/modules/sqs/sqs-listener.consumer.ts`, case `VENUE_REGISTRY_UPSERT`) gọi `processOutboxEvent(outboxId)`:

```typescript
// venue-registry.service.ts
async processOutboxEvent(outboxId: number): Promise<void> {
  const row = await this.outboxRepository.findOne({ where: { id: outboxId } });
  if (!row || row.status === DONE) return; // ← chốt idempotency: deliver trùng = no-op

  // áp dụng lên DynamoDB theo eventType, rồi đánh dấu DONE/FAILED + processedAt
  // lỗi transient: tăng attempts, throw → SQS redeliver (at-least-once)
}
```

Trạng thái đi một chiều `PENDING → PUBLISHING → DONE | FAILED`; guard `status ≠ DONE` khi ghi lỗi đảm bảo một delivery chậm không ghi đè kết quả của delivery đã hoàn thành trước đó.

**Bước 4 — Lớp bảo vệ thứ hai ở đích: conditional write.** Outbox giải quyết _delivery_, nhưng consistency ở _đích_ (slug unique toàn cục giữa 2 region) được DynamoDB tự bảo vệ bằng `ConditionExpression` (`src/services/dynamodb/venue-registry-dynamodb.service.ts`):

```typescript
new PutCommand({
  TableName,
  Item,
  ConditionExpression: "attribute_not_exists(venue_slug)", // chỉ ghi nếu slug chưa ai giữ
});
// ConditionalCheckFailedException → đọc lại item:
//   cùng venue_id  → true  (retry của chính mình, idempotent)
//   khác venue_id  → false (slug bị venue khác giữ) → outbox row FAILED + lý do
```

Hai tầng này bổ trợ nhau: outbox đảm bảo _"event chắc chắn tới nơi, ít nhất một lần"_, conditional write đảm bảo _"tới nơi nhiều lần hay tới muộn cũng không phá invariant"_. Thiếu một trong hai đều hỏng — chỉ có outbox thì retry có thể ghi đè slug của venue khác; chỉ có conditional write thì event có thể mất trước khi tới Dynamo.

### 2.4 Trade-off của cách cài trong repo

- **Độ trễ ~30s** (chu kỳ cron) — chấp nhận được vì tạo/đổi slug venue là thao tác hiếm, không cần real-time.
- **Polling tốn một query mỗi 30s** trên bảng gần như luôn rỗng — rẻ, đổi lấy việc không cần thêm hạ tầng (Debezium/DMS).
- **Bảng outbox lớn dần** — row DONE giữ lại làm audit trail; nếu volume tăng cần job dọn định kỳ (hiện chưa cần).
- **`slugExists()` chỉ là best-effort UX check** — kết quả có thể stale ngay khi trả về; guard thật là conditional PutItem. Đừng bao giờ coi check-then-act qua network là đủ.

---

## 3. Change Data Capture (CDC)

### 3.1 Lý thuyết

Ý tưởng cốt lõi trong một câu: **thay vì application tự ghi event, hạ tầng đọc trực tiếp transaction log của DB (nơi mọi thay đổi đã-commit buộc phải đi qua) và biến mỗi thay đổi thành một event.**

Mọi DB ghi thay đổi vào log trước khi áp dụng (Postgres: WAL, MySQL: binlog, DynamoDB/Mongo: Streams/oplog). CDC tool "tail" log này:

```
App ──INSERT/UPDATE/DELETE──▶ Postgres ──WAL──▶ Debezium ──▶ Kafka/Kinesis ──▶ Consumers
                                                (CDC connector)
```

Công cụ phổ biến:

- **Debezium** — connector Kafka Connect, đọc WAL Postgres qua logical replication, binlog MySQL, oplog Mongo.
- **AWS DMS** — managed CDC giữa RDS ↔ Kinesis/S3/DB khác.
- **DynamoDB Streams** — CDC tích hợp sẵn của DynamoDB: mỗi thay đổi item phát ra stream record, trigger Lambda được. (Global Table của DynamoDB replicate giữa region cũng chạy trên chính cơ chế stream này.)
- **Postgres logical replication** — dùng trực tiếp `pgoutput`/`wal2json` không cần Debezium.

Tính chất:

- **Không mất thay đổi** — commit rồi thì chắc chắn có trong log; không cần sửa code application (kể cả app legacy, kể cả người sửa tay qua SQL console cũng bắt được).
- **Độ trễ thấp** — push từ log, thường sub-second, so với chu kỳ poll của outbox.
- **Event là row-level diff** (before/after image), không phải business event — consumer thấy "row venues đổi cột slug" chứ không thấy "venue X đổi tên". Muốn ngữ nghĩa nghiệp vụ phải suy ngược, hoặc kết hợp với outbox (xem 3.3).
- **Hạ tầng nặng hơn** — cần vận hành connector, quản lý replication slot (slot Postgres bị bỏ quên sẽ giữ WAL không cho xóa → đầy disk), xử lý schema evolution.

### 3.2 Nếu venue-registry dùng CDC thì trông thế nào?

Phương án tương đương với flow hiện tại:

```
INSERT venues (Postgres)
   │  WAL
   ▼
Debezium / DMS  ──▶  Kinesis/SQS  ──▶  Consumer ghi DynamoDB
```

Application chỉ cần insert venue — không cần bảng outbox, không cần cron relay. Đổi lại: phải vận hành connector trên RDS cả 2 region, và consumer phải tự suy ra "slug đổi" từ diff của row `venues` (so sánh before/after cột `slug`), thay vì nhận event `RENAME` có sẵn `previousSlug` như outbox hiện tại.

Một chỗ CDC _đã_ hiện diện gián tiếp trong kiến trúc này: `venue-registry` là **DynamoDB Global Table**, và việc replicate AU ↔ UK giữa các region chính là AWS chạy CDC (DynamoDB Streams) hộ mình.

### 3.3 Kết hợp hay nhất của cả hai: Outbox + CDC relay

Hai pattern không loại trừ nhau. Biến thể được Debezium khuyến nghị (outbox event router): **vẫn ghi bảng outbox trong transaction** (giữ được business event có ngữ nghĩa, có `eventType`, `previousSlug`...), nhưng **relay bằng CDC tail bảng outbox** thay vì cron poll — được cả độ trễ sub-second lẫn event ngữ nghĩa, bỏ được poller. Repo hiện chọn cron poll vì đơn giản và 30s là quá đủ cho tần suất thay đổi slug; nếu sau này cần độ trễ thấp, chỉ cần thay tầng relay, phần ghi outbox và consumer giữ nguyên.

---

## 4. So sánh nhanh

| Tiêu chí                                          | Transactional Outbox (polling)       | CDC (log-based)                     |
| ------------------------------------------------- | ------------------------------------ | ----------------------------------- |
| Tầng cài đặt                                      | Application code                     | Hạ tầng (connector)                 |
| Loại event                                        | Business event (UPSERT, RENAME...)   | Row-level diff (before/after)       |
| Độ trễ                                            | Chu kỳ poll (repo: ~30s)             | Sub-second                          |
| Sửa code app                                      | Có (ghi outbox trong transaction)    | Không                               |
| Bắt được thay đổi ngoài app (SQL tay, app legacy) | Không                                | Có                                  |
| Hạ tầng thêm                                      | Bảng outbox + cron                   | Connector, replication slot, broker |
| Delivery guarantee                                | At-least-once                        | At-least-once                       |
| Yêu cầu consumer                                  | Idempotent (bắt buộc)                | Idempotent (bắt buộc)               |
| Rủi ro vận hành đặc thù                           | Row kẹt PUBLISHING (cần cron cứu hộ) | Replication slot giữ WAL → đầy disk |

Chọn thế nào:

- **Outbox** khi: cần event mang ngữ nghĩa nghiệp vụ, volume thấp, muốn zero hạ tầng mới, chấp nhận trễ vài chục giây. → đúng profile của venue-registry.
- **CDC** khi: cần trễ thấp, cần bắt mọi thay đổi kể cả ngoài app, hoặc không sửa được code nguồn (legacy). Điển hình: sync search index, cache invalidation, data lake ingestion, replicate sang analytics DB.
- **Outbox + CDC relay** khi: cần cả hai — ngữ nghĩa nghiệp vụ và trễ thấp.

Bất kể chọn gì, ba thứ **luôn bắt buộc**: consumer idempotent (at-least-once là mặc định của mọi phương án), cơ chế thu hồi claim/retry cho message kẹt, và — nếu đích có invariant như unique slug — một lớp guard ở chính đích (conditional write) thay vì tin vào check-then-act phía nguồn.
