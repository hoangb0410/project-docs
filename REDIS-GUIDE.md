# Redis — Cấu trúc dữ liệu, lệnh và các pattern sử dụng thực tế

Tài liệu tổng hợp kiến thức Redis cần thiết để đọc hiểu và thiết kế hệ thống backend, đúc kết từ một codebase sử dụng Redis ở bảy vai trò khác nhau: cache, khóa phân tán, buffer trung gian, bộ đếm tiến độ, rate limiting, pub/sub cho WebSocket, và nền tảng cho hệ thống queue (Bull). Mỗi cấu trúc dữ liệu được trình bày theo thứ tự: bản chất → lệnh chính → tình huống sử dụng thực tế.

---

## 1. Redis là gì — mô hình trong một câu

Redis là một **database NoSQL dạng key-value hoạt động trong RAM** (định nghĩa chính thức: _in-memory data structure store, used as a database, cache, and message broker_). Điểm phân biệt nó với các database key-value khác: value không chỉ là chuỗi mà có thể là list, set, hash, sorted set — mỗi cấu trúc có bộ lệnh thao tác riêng chạy ngay trên server, nên thay vì "lấy dữ liệu về sửa rồi ghi lại", ta ra lệnh cho Redis tự sửa tại chỗ.

Là database nghĩa là nó có cả cơ chế bền hóa dữ liệu xuống đĩa (RDB snapshot / AOF log) và có thể làm nơi lưu trữ chính. Tuy nhiên trong đa số hệ thống backend — bao gồm codebase này — Redis được dùng cho **dữ liệu trạng thái ngắn hạn** (cache, lock, buffer, counter, queue), còn dữ liệu gốc nằm ở database quan hệ; vì vậy mọi thiết kế trong tài liệu này đều xuất phát từ giả định "dữ liệu trong Redis có thể biến mất".

Ba đặc tính quyết định mọi cách dùng Redis:

1. **Toàn bộ dữ liệu nằm trong RAM** → đọc/ghi ở mức trăm nghìn ops/giây, độ trễ dưới mili-giây; đổi lại dung lượng đắt và hữu hạn (`maxmemory`).
2. **Xử lý lệnh trên một thread duy nhất** → các lệnh được thực thi tuần tự, do đó **mỗi lệnh đơn là atomic tuyệt đối** — hai client cùng `INCR` một key không bao giờ giẫm nhau. Hệ quả ngược: một lệnh chậm (quét toàn bộ keyspace, script Lua dài) chặn _tất cả_ client khác.
3. **Key có vòng đời (TTL)** → Redis tự xóa key hết hạn, là nền tảng cho cache, lock tự nhả và marker tạm thời.

---

## 2. Vòng đời key và quy ước đặt tên

### TTL

| Lệnh                      | Ý nghĩa                                                       |
| ------------------------- | ------------------------------------------------------------- |
| `EXPIRE key seconds`      | Gắn thời gian sống cho key có sẵn                             |
| `SETEX key seconds value` | Ghi value kèm TTL trong một lệnh (atomic)                     |
| `TTL key`                 | Xem còn sống bao lâu (`-1` = vĩnh viễn, `-2` = không tồn tại) |
| `PERSIST key`             | Gỡ TTL, key sống vĩnh viễn                                    |

(Biến thể mili-giây: `PEXPIRE`, `PSETEX`, `PTTL`; hẹn theo mốc thời gian tuyệt đối: `EXPIREAT`.)

### Lệnh keyspace chung (áp dụng cho key mọi kiểu)

| Lệnh                | Ý nghĩa                                                       |
| ------------------- | ------------------------------------------------------------- |
| `DEL key...`        | Xóa ngay (chặn nếu value rất lớn)                             |
| `UNLINK key...`     | Xóa "trả góp" ở background — chọn thay DEL cho key to         |
| `EXISTS key...`     | Đếm bao nhiêu key tồn tại                                     |
| `TYPE key`          | Key đang giữ value kiểu gì (string/list/set/hash/zset/stream) |
| `RENAME key newkey` | Đổi tên key                                                   |
| `SCAN`              | Duyệt keyspace theo mẻ (mục 6)                                |
| `MEMORY USAGE key`  | Key này tốn bao nhiêu bytes                                   |

Nguyên tắc: **mọi key mang tính tạm thời (marker, lock, cache, buffer) bắt buộc có TTL**. Key không TTL chỉ dành cho dữ liệu chủ đích sống lâu. Key rác không TTL tích tụ dần là nguyên nhân phổ biến nhất khiến Redis đầy bộ nhớ.

### Eviction

Khi chạm `maxmemory`, Redis xử lý theo policy cấu hình: `noeviction` (từ chối ghi — ứng dụng nhận lỗi), hoặc các biến thể LRU/LFU (tự đuổi key ít dùng). Với Redis vừa làm cache vừa giữ **trạng thái pipeline** (lock, counter, buffer), eviction tự động rất nguy hiểm — key trạng thái bị đuổi giữa chừng đồng nghĩa pipeline mất trí nhớ. Bài học thực tế: một sự cố OOM Redis từng xảy ra do hàng chục nghìn key/timer tích tụ, và cách phòng là TTL nghiêm túc + theo dõi bộ nhớ (`MEMORY USAGE key` để đo một key cụ thể).

### Đặt tên key

Quy ước phổ biến (và dùng trong codebase): phân cấp bằng dấu hai chấm, đi từ phạm vi rộng đến hẹp, kèm tiền tố ứng dụng để nhiều môi trường dùng chung một Redis không giẫm nhau (ioredis hỗ trợ `keyPrefix` tự gắn):

```
job:{jobId}:customerBuffer            — buffer của một phiên sync
job:{jobId}:page:137:fetched          — marker trang 137 đã fetch
venue:48:resdiary:failedPages         — sổ trang lỗi của venue 48
lock:buffer:{jobId}                   — khóa xúc buffer
avail:{venueId}:{date}:{party}:v{n}   — cache có số phiên bản trong tên
```

Tên key chứa đủ tọa độ để tự mô tả, và version nằm ngay trong tên là nền của pattern invalidation ở mục 6.

---

## 3. Các cấu trúc dữ liệu và lệnh

Trước khi vào từng loại, ba điều nền tảng về mô hình key-value của Redis:

- **Key luôn luôn là chuỗi** (binary-safe, tối đa 512MB nhưng thực tế nên ngắn). Không tồn tại "key kiểu số" hay "key kiểu list" — mọi phân loại dưới đây là **kiểu của value**.
- **Kiểu của value gắn chặt với key**: key nào đã giữ List thì mọi lệnh String lên key đó trả lỗi `WRONGTYPE` (Redis không tự ép kiểu). Lệnh `TYPE key` cho biết key đang giữ kiểu gì — công cụ soi đầu tiên khi vào `redis-cli`.
- **Không có kiểu số**: "số" trong Redis là chuỗi trông giống số; `INCR` parse chuỗi thành số nguyên, tăng, rồi ghi lại chuỗi. Vì vậy `INCR` lên value `"abc"` trả lỗi runtime chứ không phải lỗi kiểu khai báo.

Mỗi bảng dưới đây liệt kê đủ nhóm lệnh chuẩn của loại đó; các lệnh codebase dùng trực tiếp được ghi rõ tình huống ở cột cuối, lệnh còn lại để nhận diện khi đọc code/tra cứu.

### 3.1. String — mặc định cho mọi thứ

Value là một chuỗi bytes bất kỳ (thường là JSON hoặc số). Đây là cấu trúc đa dụng nhất — cache, marker, counter, lock đều là String.

| Lệnh                                                  | Ý nghĩa                                                                                                  | Dùng thật ở                                                                 |
| ----------------------------------------------------- | -------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------- |
| `GET` / `SET`                                         | Đọc / ghi                                                                                                | Cache JSON kết quả tính toán                                                |
| `SET ... EX ttl / PX ms / NX / XX / GET`              | Các option của SET: kèm TTL giây/mili-giây, chỉ-khi-chưa-có (NX), chỉ-khi-đã-có (XX), trả value cũ (GET) | `SET key val EX ttl NX` = khóa phân tán chuẩn (mục 7.1)                     |
| `SETEX key ttl value`                                 | Ghi kèm TTL (dạng cũ của `SET ... EX`)                                                                   | Marker idempotency "trang N đã fetch" (TTL 24h), PKCE verifier (TTL 5 phút) |
| `SETNX key value`                                     | Chỉ ghi nếu key **chưa tồn tại** — trả 1 nếu ghi được                                                    | Chống enqueue trùng một batch                                               |
| `MSET` / `MGET k1 k2...`                              | Ghi / đọc nhiều key một lệnh                                                                             | — (đọc nhiều key codebase dùng pipeline)                                    |
| `INCR` / `DECR` / `INCRBY` / `DECRBY` / `INCRBYFLOAT` | Tăng / giảm số atomic, key chưa có coi như 0                                                             | Bộ đếm số batch, bộ đếm record đã xử lý, version key                        |
| `APPEND key value`                                    | Nối vào cuối chuỗi                                                                                       | —                                                                           |
| `STRLEN`                                              | Độ dài chuỗi (bytes)                                                                                     | —                                                                           |
| `GETRANGE` / `SETRANGE`                               | Đọc / ghi đè một đoạn theo offset                                                                        | —                                                                           |
| `GETDEL` / `GETEX`                                    | Đọc-rồi-xóa / đọc-rồi-đổi-TTL trong một lệnh                                                             | —                                                                           |

Lưu ý quan trọng: `INCR` atomic nghĩa là 20 worker cùng gọi thì kết quả vẫn đúng tổng — không cần lock. Đây là lý do bộ đếm tiến độ đặt ở Redis chứ không ở biến trong process (nhiều process, nhiều instance ECS).

### 3.2. List — hàng đợi / băng chuyền

Danh sách hai đầu (deque). Push/pop ở hai đầu đều O(1).

| Lệnh                                          | Ý nghĩa                                                                    | Dùng thật ở                                             |
| --------------------------------------------- | -------------------------------------------------------------------------- | ------------------------------------------------------- |
| `LPUSH` / `RPUSH`                             | Thêm vào đầu / cuối (nhận nhiều value một lệnh)                            | Job fetch đẩy nguyên trang JSON (100 record) vào buffer |
| `LPUSHX` / `RPUSHX`                           | Chỉ push nếu list **đã tồn tại**                                           | —                                                       |
| `LPOP` / `RPOP [count]`                       | Lấy ra và xóa khỏi đầu / cuối                                              | Xúc tối đa 5 trang khỏi buffer để đóng batch            |
| `BLPOP` / `BRPOP key timeout`                 | Như LPOP/RPOP nhưng **chờ** (blocking) đến khi có phần tử hoặc hết timeout | — (nền của worker-queue kiểu chờ việc)                  |
| `LLEN`                                        | Đếm số phần tử                                                             | "Ngó khay": buffer đủ dày chưa?                         |
| `LRANGE key start stop`                       | Đọc một dải (không xóa); `0 -1` = cả list                                  | Xem nội dung buffer khi debug                           |
| `LINDEX key i` / `LSET key i value`           | Đọc / ghi đè phần tử tại vị trí i (O(N))                                   | —                                                       |
| `LREM key count value`                        | Xóa phần tử theo giá trị                                                   | Gỡ một mục cụ thể khỏi danh sách                        |
| `LTRIM key start stop`                        | Cắt list, chỉ giữ dải chỉ định — cách giữ "N phần tử mới nhất"             | —                                                       |
| `LINSERT key BEFORE/AFTER pivot value`        | Chèn cạnh một phần tử có sẵn                                               | —                                                       |
| `LMOVE src dst LEFT/RIGHT` (thay `RPOPLPUSH`) | Chuyển phần tử giữa 2 list trong một lệnh atomic                           | — (pattern hàng đợi có "đang xử lý")                    |

Mô hình sử dụng điển hình: **producer LPUSH, consumer LPOP** — chính là băng chuyền trung gian tách rời tầng fetch và tầng ghi DB trong pipeline sync. Điểm cần nhớ: `LPOP` mặc định lấy một phần tử; muốn lấy N phần tử atomic trên Redis cũ phải bọc vòng lặp `lpop` trong script Lua (codebase làm đúng như vậy).

### 3.3. Set — tập hợp không trùng lặp

Tập các chuỗi duy nhất, không thứ tự. Thêm phần tử đã có = không làm gì.

| Lệnh                                            | Ý nghĩa                                                                | Dùng thật ở                                                      |
| ----------------------------------------------- | ---------------------------------------------------------------------- | ---------------------------------------------------------------- |
| `SADD key member...`                            | Thêm (tự khử trùng)                                                    | Ghi sổ trang fetch lỗi: `SADD venue:48:resdiary:failedPages 137` |
| `SISMEMBER` / `SMISMEMBER`                      | Một / nhiều phần tử có trong tập không? (O(1))                         | Kiểm tra một phần tử đã được xử lý chưa                          |
| `SMEMBERS`                                      | Lấy toàn bộ (O(N) — tập lớn dùng `SSCAN`)                              | Endpoint admin đọc danh sách trang lỗi để chạy lại               |
| `SSCAN key cursor`                              | Duyệt tập lớn theo mẻ, không chặn server                               | —                                                                |
| `SCARD`                                         | Đếm số phần tử                                                         | Báo cáo số trang lỗi                                             |
| `SREM`                                          | Xóa phần tử                                                            | Gỡ trang khỏi sổ sau khi retry thành công                        |
| `SPOP key [count]`                              | Lấy ra và xóa **ngẫu nhiên**                                           | — (rút thăm, phân việc ngẫu nhiên)                               |
| `SRANDMEMBER key [count]`                       | Đọc ngẫu nhiên, không xóa                                              | —                                                                |
| `SMOVE src dst member`                          | Chuyển phần tử giữa 2 tập atomic                                       | —                                                                |
| `SUNION` / `SINTER` / `SDIFF` (+`...STORE dst`) | Hợp / giao / hiệu của nhiều tập, biến thể STORE ghi kết quả ra key mới | — (giao 2 tập tag, khác biệt giữa 2 lần chụp)                    |

Set là câu trả lời cho mọi nhu cầu "danh sách không được trùng + kiểm tra tồn tại nhanh" — dedup, sổ ghi lỗi, đánh dấu tập đã thấy.

### 3.4. Hash — object nhiều field dưới một key

Một key chứa nhiều cặp field-value, như một object JSON phẳng một tầng.

| Lệnh                                    | Ý nghĩa                                                             |
| --------------------------------------- | ------------------------------------------------------------------- |
| `HSET key field value [field value...]` | Ghi một hoặc nhiều field (thay thế `HMSET` cũ)                      |
| `HSETNX key field value`                | Chỉ ghi nếu field chưa tồn tại                                      |
| `HGET key field` / `HMGET key f1 f2...` | Đọc một / nhiều field                                               |
| `HGETALL key`                           | Lấy toàn bộ field-value (O(N) — hash lớn dùng `HSCAN`)              |
| `HDEL` / `HEXISTS`                      | Xóa / kiểm tra một field                                            |
| `HLEN`                                  | Đếm số field                                                        |
| `HKEYS` / `HVALS`                       | Lấy riêng danh sách field / danh sách value                         |
| `HINCRBY key field n` / `HINCRBYFLOAT`  | Tăng số atomic trên **một field** — nhiều bộ đếm cùng chung một key |
| `HRANDFIELD key [count]`                | Lấy field ngẫu nhiên                                                |
| `HSCAN`                                 | Duyệt hash lớn theo cursor, không chặn server                       |

Khi nào chọn Hash thay vì nhiều key String: các field cùng thuộc một thực thể, cùng vòng đời (TTL đặt trên cả hash, không đặt được trên từng field), và muốn đọc cả cụm bằng một `HGETALL`. Trong codebase, Hash dùng cho trạng thái các cron job nhiều thuộc tính. Ngược lại, nếu từng field cần TTL riêng → tách thành key String riêng.

### 3.5. Sorted Set — tập có điểm số (dùng ngầm qua Bull)

Mỗi member là chuỗi duy nhất kèm một `score` số thực; Redis giữ tập luôn sắp theo score. Là cấu trúc cho mọi bài toán "xếp hạng / lấy theo khoảng / hẹn giờ".

| Lệnh                                                         | Ý nghĩa                                                                                           |
| ------------------------------------------------------------ | ------------------------------------------------------------------------------------------------- |
| `ZADD key score member` (option `NX/XX/GT/LT`)               | Thêm hoặc cập nhật score (GT/LT: chỉ cập nhật nếu lớn/nhỏ hơn score cũ)                           |
| `ZSCORE key member` / `ZMSCORE`                              | Đọc score của một / nhiều member                                                                  |
| `ZINCRBY key n member`                                       | Cộng vào score atomic — nền của leaderboard                                                       |
| `ZCARD` / `ZCOUNT key min max`                               | Đếm cả tập / đếm trong khoảng score                                                               |
| `ZRANGE key start stop [REV] [BYSCORE/BYLEX] [WITHSCORES]`   | Lấy theo thứ hạng hoặc theo khoảng score (dạng hợp nhất mới, thay `ZRANGEBYSCORE`/`ZREVRANGE` cũ) |
| `ZRANK` / `ZREVRANK key member`                              | Thứ hạng của member (từ thấp / từ cao)                                                            |
| `ZREM` / `ZREMRANGEBYSCORE` / `ZREMRANGEBYRANK`              | Xóa theo member / theo khoảng score / theo khoảng hạng                                            |
| `ZPOPMIN` / `ZPOPMAX [count]` (+bản blocking `BZPOPMIN/MAX`) | Lấy ra và xóa member score nhỏ nhất / lớn nhất                                                    |
| `ZUNIONSTORE` / `ZINTERSTORE`                                | Hợp / giao nhiều sorted set, ghi ra key mới                                                       |
| `ZSCAN`                                                      | Duyệt tập lớn theo cursor                                                                         |

Codebase không gọi trực tiếp, nhưng **Bull xây toàn bộ cơ chế delayed job và priority trên Sorted Set**: job hẹn giờ được `ZADD` với score = timestamp chạy, worker lấy job đến hạn bằng truy vấn theo khoảng score. Hiểu điều này để biết "job delay 30 giây" thực chất nằm ở đâu khi vào `redis-cli` soi.

### 3.6. Pub/Sub — phát thanh không lưu trữ

`PUBLISH channel message` / `SUBSCRIBE channel`: gửi tin cho mọi subscriber đang nghe _tại thời điểm đó_; không lưu, không retry — ai không online là mất tin. Dùng thật ở: **Socket.IO Redis adapter** — nhiều instance API cùng chạy, một instance emit sự kiện WebSocket thì adapter PUBLISH qua Redis để các instance khác chuyển tiếp cho client đang kết nối với chúng. Cần một connection riêng cho subscribe (codebase `duplicate()` connection vì connection đã SUBSCRIBE không chạy được lệnh thường).

Phân biệt với queue: Pub/Sub là "loa phát thanh" (mất là mất), List/Bull là "hòm thư" (nằm đó chờ xử lý). Nhu cầu cần cả hai tính chất → Redis Streams (codebase chưa dùng).

### 3.7. Các kiểu value khác — biết để nhận diện

Codebase không dùng, nhưng chúng tồn tại và có chỗ đứng riêng:

- **Streams** (`XADD`, `XREAD`, `XREADGROUP`, `XACK`): log sự kiện append-only có lưu trữ + consumer group có ack/retry — như một Kafka thu nhỏ trong Redis; là lựa chọn khi cần pub/sub _không mất tin_.
- **Bitmap** (`SETBIT`, `GETBIT`, `BITCOUNT`): thao tác từng bit trên một String — đánh dấu hiện diện theo id (ví dụ user nào đã online hôm nay) với chi phí 1 bit/id.
- **HyperLogLog** (`PFADD`, `PFCOUNT`): đếm số phần tử _khác nhau_ xấp xỉ (sai số ~0,8%) với bộ nhớ cố định 12KB bất kể hàng tỷ phần tử — đếm unique visitor mà không lưu danh sách.
- **Geospatial** (`GEOADD`, `GEOSEARCH`): lưu tọa độ và truy vấn "trong bán kính X km" — thực chất là Sorted Set với score mã hóa geohash.

---

## 4. Ba cấp độ atomic — lệnh đơn, Lua script, pipeline

Đây là phần quan trọng nhất khi dùng Redis trong hệ song song.

### Cấp 1 — lệnh đơn: luôn atomic

`INCR`, `SETNX`, `LPUSH`... tự thân an toàn trước mọi race. Nếu bài toán giải được bằng một lệnh, không cần gì thêm.

### Cấp 2 — Lua script (`EVAL`): chuỗi lệnh atomic

Vấn đề kinh điển: logic dạng **"đọc → quyết định → ghi"** (check-then-act) bằng hai lệnh rời sẽ dính race — giữa lệnh đọc và lệnh ghi, client khác có thể chen vào. Giải pháp: gói cả chuỗi vào một script Lua; Redis chạy script **như một lệnh duy nhất**, không gì chen ngang được.

Ba ví dụ thật trong codebase:

```lua
-- (1) Nhả lock an toàn: chỉ xóa nếu lock vẫn là của mình
if redis.call("get", KEYS[1]) == ARGV[1] then
  return redis.call("del", KEYS[1])
else
  return 0
end
```

```lua
-- (2) LPOP N phần tử atomic (Redis cũ không có LPOP count)
local result = {}
for i = 1, ARGV[1] do
  local value = redis.call('lpop', KEYS[1])
  if value then table.insert(result, value) else break end
end
return result
```

Ví dụ (3) là rate limiter: script kiểm tra key block → `INCR` bộ đếm → nếu chưa có TTL thì `PEXPIRE` → nếu vượt ngưỡng thì đặt key block — bốn bước "đọc-quyết-ghi" lồng nhau, chỉ đúng khi atomic toàn khối. Tương tự, pipeline sync dùng một script "đánh dấu batch đã xử lý + tăng 3 bộ đếm" để batch bị giao lại không bị đếm hai lần.

Lua đủ quan trọng để có phần riêng: mục 5 trình bày đầy đủ cơ chế, cú pháp, ví dụ và các bẫy.

### Cấp 3 — Pipeline: gộp round-trip, KHÔNG atomic

`pipeline()` gửi một loạt lệnh trong một chuyến mạng thay vì mỗi lệnh một round-trip — tăng throughput hàng chục lần khi ghi hàng loạt (codebase dùng khi ghi batch nhiều key). Nhưng các lệnh trong pipeline **có thể bị lệnh của client khác xen giữa** — pipeline là tối ưu hiệu năng, không phải công cụ đúng đắn. Cần atomic → Lua (hoặc `MULTI/EXEC`, ít dùng hơn vì Lua linh hoạt hơn).

Bảng chọn nhanh:

| Nhu cầu                                | Công cụ     |
| -------------------------------------- | ----------- |
| Một thao tác đơn                       | Lệnh thường |
| Đọc-rồi-quyết-rồi-ghi phải liền mạch   | Lua script  |
| Ghi rất nhiều key, không cần liền mạch | Pipeline    |

---

## 5. Lua script trong Redis

### Là gì

Lua script là một **đoạn chương trình nhỏ** (viết bằng ngôn ngữ Lua) mà client gửi sang cho Redis, và **Redis tự chạy nó ngay bên trong server**. Với Redis, cả đoạn script được đối xử **như một lệnh duy nhất** — giống như bạn tự chế thêm được một lệnh mới mà Redis chưa có.

### Đặc điểm

- **Atomic tuyệt đối**: trong lúc script chạy, không một lệnh nào của client khác chen vào giữa được. Đọc xong quyết định xong ghi xong — liền một mạch.
- **Chạy tại chỗ dữ liệu**: các bước trong script không tốn round-trip mạng; 5 lệnh trong script nhanh hơn hẳn 5 lệnh gửi rời.
- **Nhưng cũng chặn tuyệt đối**: Redis chỉ có một thread — script chạy 1 giây nghĩa là _mọi_ client khác treo 1 giây.
- **Không nhớ gì giữa các lần chạy**: biến trong script chết khi script kết thúc; trạng thái bền phải nằm trong key Redis.
- **Không được ra ngoài**: script chỉ được gọi lệnh Redis và tính toán thuần — không gọi API, không đọc file, không sleep.

### Mục đích — khi nào cần đến nó

Vấn đề gốc: logic dạng **"đọc → quyết định → ghi"** viết bằng hai lệnh rời sẽ dính race — giữa lệnh đọc và lệnh ghi, client khác kịp chen vào làm dữ liệu đổi rồi. Lua gói cả chuỗi thành một khối không thể chen ngang. Ba tình huống dùng điển hình:

1. **Check-then-act**: "chỉ xóa lock nếu nó vẫn là của mình", "chỉ trừ kho nếu còn đủ hàng".
2. **Cập nhật nhiều key phải nhất quán**: "đánh dấu batch đã xử lý + tăng 3 bộ đếm" — không được để lệnh nào thành công lệnh nào không.
3. **Vá lệnh Redis còn thiếu**: bản Redis cũ không có `LPOP count` → viết vòng lặp lpop nhỏ trong script.

### Cú pháp

```
EVAL <script> <numkeys> <key1> <key2>... <arg1> <arg2>...
```

| Tham số   | Ý nghĩa                                                                                                                          |
| --------- | -------------------------------------------------------------------------------------------------------------------------------- |
| `script`  | Chuỗi chứa code Lua                                                                                                              |
| `numkeys` | Số lượng tham số phía sau là **key**. Redis cần biết ranh giới này để route đúng node khi chạy cluster — khai sai là bom nổ chậm |
| `key1...` | Các key script sẽ đụng tới → vào script qua mảng `KEYS[1]`, `KEYS[2]`...                                                         |
| `arg1...` | Tham số dữ liệu thuần → vào script qua mảng `ARGV[1]`, `ARGV[2]`...                                                              |

Bên trong script, gọi lệnh Redis bằng:

- `redis.call('GET', KEYS[1])` — lỗi thì script dừng ngay, lỗi ném về client.
- `redis.pcall(...)` — lỗi trả về dạng object để script tự xử lý rồi đi tiếp.

Qua ioredis (wrapper trong codebase): `redis.eval(script, numKeys, key1, ..., arg1, ...)` — đúng thứ tự tham số như trên.

### Ví dụ đơn giản — trừ kho nếu còn đủ hàng

Bài toán: còn `stock:item:42 = 5` sản phẩm, khách đặt 3. Nếu viết `GET` rồi `DECRBY` bằng hai lệnh rời, hai khách cùng đặt có thể cùng thấy "còn 5" và cùng trừ — kho âm. Gói vào script:

```lua
local stock = tonumber(redis.call('GET', KEYS[1]) or '0')
if stock >= tonumber(ARGV[1]) then
  redis.call('DECRBY', KEYS[1], ARGV[1])
  return 1   -- trừ thành công
else
  return 0   -- không đủ hàng
end
```

Gọi:

```
EVAL "<script trên>" 1 stock:item:42 3
         numkeys ────┘      │         └── ARGV[1] = "3"
                            └── KEYS[1] = "stock:item:42"
```

Hai khách cùng gọi thì Redis chạy script lần lượt từng người — người sau đọc được số kho _sau khi_ người trước đã trừ. Race biến mất, không cần lock. (Hai script thật trong codebase — nhả lock so token và LPOP N phần tử — nằm ở mục 4.)

### Các lưu ý

1. **Không viết script xử lý quá lâu** — không vòng lặp lồng nhau phức tạp, không quét tập dữ liệu lớn. Vì script chặn toàn server: vòng lặp `lpop` 5 phần tử là tốt; vòng lặp quét list 100 nghìn phần tử là sự cố production. Chạy quá 5 giây (mặc định), server trả lỗi `BUSY` cho mọi client, và `SCRIPT KILL` chỉ cứu được khi script **chưa ghi gì** — đã ghi rồi thì chỉ còn cách chờ hoặc tắt server bỏ dữ liệu. "Script ngắn" là luật, không phải khuyến nghị.
2. **Mọi `ARGV` là chuỗi** — muốn so sánh/tính toán số phải bọc `tonumber(ARGV[1])`. Quên là bug so sánh chuỗi âm thầm.
3. **Số thực trả về bị cắt thành số nguyên** (`3.99` → `3`) — cần số thực thì trả dạng chuỗi.
4. **Key không tồn tại: `redis.call('GET', ...)` trả `false`**, không phải `nil` — viết `if value then` vẫn đúng, so sánh tường minh với `nil` thì sai.
5. **Table trả về client thành array, gặp phần tử `nil` là cắt cụt tại đó.**
6. **Khai `numkeys` đúng, key đi đường KEYS, dữ liệu đi đường ARGV** — trộn lẫn vẫn chạy trên Redis đơn nhưng vỡ khi chuyển cluster.
7. **Không random, không lấy giờ hệ thống trong script** — cần timestamp thì truyền từ ứng dụng vào qua `ARGV`.
8. **Script dùng thường xuyên nên nạp trước bằng `SCRIPT LOAD`** rồi gọi qua `EVALSHA <sha1>` — tránh gửi cả đoạn code qua mạng mỗi lần; client gặp `NOSCRIPT` (server mới restart) thì fallback về `EVAL`. Codebase gọi `eval` trực tiếp — chấp nhận được vì script ngắn, tần suất thưa.
9. **Đừng nhét business logic vào script** — Lua trong Redis là để bảo vệ tính atomic của vài thao tác dữ liệu, không phải nơi viết nghiệp vụ. Logic dài, gọi API, xử lý phức tạp → kéo về ứng dụng.

Bảng quyết định nhanh:

| Tình huống                                          | Quyết định                |
| --------------------------------------------------- | ------------------------- |
| Check-then-act (lock release, trừ kho có điều kiện) | Lua — đây là chỗ của nó   |
| Cập nhật nhiều key phải nhất quán với nhau          | Lua                       |
| Thiếu một lệnh Redis mong muốn                      | Lua vá lệnh, vòng lặp nhỏ |
| Business logic, vòng lặp dữ liệu lớn, gọi API       | Không — kéo về ứng dụng   |
| Chỉ cần giảm round-trip, các lệnh độc lập nhau      | Pipeline, khỏi Lua        |

---

## 6. Duyệt keyspace: SCAN, không bao giờ KEYS

`KEYS pattern` quét toàn bộ keyspace trong **một lệnh chặn** — trên Redis production hàng triệu key, đây là lệnh đóng băng cả hệ thống. Thay bằng `SCAN`:

```
SCAN cursor MATCH "venue:48:*" COUNT 500
```

SCAN trả về từng mẻ nhỏ kèm cursor để gọi tiếp, server không bị chặn giữa các mẻ (đổi lại: kết quả không phải snapshot nhất quán — key thêm/xóa trong lúc duyệt có thể xuất hiện hoặc không). Codebase bọc sẵn `deletePattern(pattern)` = vòng lặp SCAN + DEL từng mẻ — đây là cách xóa cache theo prefix an toàn. Tương tự có `HSCAN` cho hash lớn.

---

## 7. Các pattern thực chiến

### 7.1. Khóa phân tán (distributed lock)

Bài toán: nhiều worker/instance cùng muốn làm một việc chỉ được làm một lần (xúc buffer, chạy cron, tính toán cache). Công thức chuẩn:

```
Giành:  SET lock:xxx {token-ngẫu-nhiên} EX {ttl} NX     → OK = mình được làm
Nhả:    Lua script so token rồi mới DEL (ví dụ mục 4)
```

Ba chi tiết bắt buộc, thiếu một là sai:

- **NX + EX trong một lệnh** — tách SETNX rồi EXPIRE thành hai lệnh thì crash giữa chừng để lại lock vĩnh viễn.
- **TTL** — người giữ lock chết thì lock tự nhả sau TTL; chọn TTL theo thời gian tối đa của việc (codebase: 10s cho xúc buffer, 300s cho batch, 600s cho cron).
- **Token riêng khi nhả** — không có nó, worker chạy quá TTL sẽ DEL nhầm lock mà người khác vừa giành được.

Hệ quả cần chấp nhận: lock TTL là **best-effort** — việc chạy lâu hơn TTL thì có thể hai bên cùng làm; vì vậy lock chỉ dùng để _giảm_ trùng lặp, còn tính đúng đắn cuối cùng phải do tầng dữ liệu bảo đảm (unique constraint, idempotency).

### 7.2. Marker idempotency

`SETEX marker ttl 1` sau khi hoàn thành một đơn vị việc; đầu handler `GET` thấy `1` thì bỏ qua. Dùng cho: trang đã fetch, webhook đã xử lý, batch đã enqueue. TTL đặt theo "khoảng thời gian mà retry còn có thể xảy ra" (24h trong pipeline sync).

### 7.3. Bộ đếm tiến độ phân tán

`INCRBY` từ mọi worker về cùng một key; kẻ nào tăng xong thấy tổng vượt ngưỡng thì kích hoạt bước kế (chuyển phase khi ≥ 95%). Kết hợp Lua khi "tăng đếm" phải đi liền "đánh dấu đã đếm" để chống đếm trùng.

### 7.4. Cache-aside với version trong tên key

Pattern cache đáng học nhất trong codebase (module tính availability):

- Tên key chứa số phiên bản: `avail:{venue}:{date}:{party}:v{version}`.
- Đọc: lấy version hiện hành (một key `version:{venue}:{date}` riêng) → ghép tên → `GET`; miss thì tính lại và `SETEX` (TTL 60s).
- **Invalidation = tăng version** (`INCR` key version), không phải đi xóa từng key cache: mọi key cũ tự thành mồ côi và chết theo TTL. Xóa O(1) bất kể bao nhiêu key liên quan, và không cần biết trước danh sách key phải xóa.
- Chống stampede: khi cache miss, các reader **giành compute-lock** (mục 7.1, TTL 15s) — một người tính, kẻ thua đọc bản cũ (key stale TTL dài hơn) thay vì cả đàn cùng dội vào DB.

### 7.5. Rate limiting

Bộ đếm `INCR` + `PEXPIRE` theo cửa sổ thời gian, vượt ngưỡng thì đặt key block — toàn bộ trong một Lua script (mục 4, ví dụ 3). Đặt ở Redis để ngưỡng được chia sẻ giữa mọi instance API, thay vì mỗi instance đếm riêng.

### 7.6. Hạ tầng đứng trên Redis mà ít ai để ý

- **Bull queue**: mỗi queue là một cụm key Redis (list job chờ, sorted set job delay, hash dữ liệu job). "Xóa job", "retry", "priority" đều là thao tác Redis.
- **Socket.IO adapter**: pub/sub giữa các instance (mục 3.6).
- Hệ quả vận hành: **flush Redis = mất sạch queue + trạng thái pipeline + cache cùng lúc** — là biện pháp cuối cùng khi sự cố, không phải thao tác dọn dẹp thông thường.

---

## 8. Checklist thực dụng khi thêm một key Redis mới

1. Key tạm thời chưa? → **có TTL chưa?**
2. Tên key có đủ tọa độ (prefix ứng dụng, phạm vi, id) và theo cấp `a:b:c` không?
3. Thao tác có dạng "đọc-rồi-quyết" không? → cần Lua, hai lệnh rời là race.
4. Value có to không (JSON cả trang dữ liệu)? → ước lượng tổng bộ nhớ ở quy mô thật, nghĩ đến `maxmemory`.
5. Nhiều instance cùng ghi? → lệnh đã atomic chưa, hay cần lock/SETNX?
6. Nếu key này **biến mất giữa chừng** (eviction, flush, TTL hết sớm) thì hệ thống hỏng kiểu gì — có tầng nào đỡ không (unique constraint, watchdog, force-complete)?
7. Cần tìm lại key theo pattern? → chỉ SCAN, không KEYS; hoặc thiết kế lại để khỏi phải quét (version key).

Câu hỏi số 6 là quan trọng nhất: Redis nhanh vì nó được phép quên — mọi thiết kế đúng đều phải trả lời được "quên thì sao".
