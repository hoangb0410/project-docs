# NestJS DI Scope & Memory Leak — lý thuyết ngắn

Cặp đôi với `LLM.md` — lý thuyết portable, ví dụ lấy từ nollie-api.

---

## 1. Ba scope của provider

| Scope | Vòng đời instance | Khi nào dùng |
|---|---|---|
| **Singleton** (mặc định) | 1 instance / cả app, tạo lúc bootstrap, sống suốt đời process | 99% trường hợp — service, repository, SDK client |
| **Request** (`Scope.REQUEST`) | 1 instance / mỗi HTTP request, GC sau khi response xong | khi provider PHẢI giữ state theo request (tenant context, request-id) và không dùng được cách khác |
| **Transient** (`Scope.TRANSIENT`) | 1 instance / mỗi **chỗ inject** | provider có state riêng per-consumer (vd logger gắn tên class) — hiếm |

```ts
@Injectable()                              // singleton (mặc định)
@Injectable({ scope: Scope.REQUEST })      // request
@Injectable({ scope: Scope.TRANSIENT })    // transient
```

## 2. Hai luật quan trọng của Request scope

1. **Scope bubbling (lây ngược)**: service A inject provider request-scoped → A tự động thành request-scoped → controller dùng A cũng vậy. Một provider request-scoped kéo **cả cây DI** ra khỏi singleton cache → mỗi request phải khởi tạo lại cả chuỗi → GC pressure, chậm.
2. **Chỉ tồn tại trong HTTP context**: worker queue (SQS/BullMQ), cron job, socket gateway **không có request** → inject provider request-scoped vào đó là lỗi runtime. Hệ nào có background worker thì request scope gần như bị loại từ vòng gửi xe.

## 3. Hợp đồng của Singleton: stateless service

Instance dùng chung cho mọi request chạy song song, nên:

- **Mọi state per-request nằm trên stack** — tham số hàm, biến cục bộ. Guard gắn `user` vào request → controller lấy ra → **truyền xuống service qua tham số** (`doThing(venueId, dto)`).
- **Cấm lưu dữ liệu request vào `this.xxx`** — vừa leak, vừa **lẫn dữ liệu giữa 2 request song song**; với multi-tenant nghĩa là lộ data chéo tenant. Đây là bug tệ nhất của nhóm này vì nó im lặng và ngẫu nhiên.
- Field của class chỉ dành cho: dependency (inject), SDK client tạo 1 lần trong constructor (DB pool, Redis, Anthropic/OpenAI client), config, và state toàn cục có chủ đích (xem mục 4).

> 📌 *nollie-api*: 100% singleton — không có `Scope.REQUEST`/`TRANSIENT` nào trong codebase; `user`/`venueId` đi qua tham số hàm; SDK client khởi tạo trong constructor.

## 4. Memory leak trong thế giới singleton — 4 vector

| Vector | Dạng | Phòng |
|---|---|---|
| Request state trên field | `this.currentUser = user` | stateless contract (mục 3) |
| Cấu trúc in-memory không trần | `private cache = new Map()` chỉ thêm không xóa → phình vô hạn | key theo tập **hữu hạn** (per provider/model, không per request/user), hoặc LRU + TTL, hoặc chuyển sang Redis |
| Listener/interval tích lũy | `setInterval` / `emitter.on()` gọi trong method (mỗi request đăng ký thêm 1 cái) | đăng ký 1 lần trong constructor / `onModuleInit`, hủy trong `onModuleDestroy` |
| Closure giữ object to | fire-and-forget promise giữ tham chiếu buffer/base64 đến khi xong | không phải leak (có kết thúc) nhưng là RAM spike — biết để không hoảng khi nhìn metric |

Quy tắc chẩn đoán: leak thật = heap **tăng đơn điệu qua nhiều giờ** và không xuống sau GC; còn răng cưa lên-xuống là bình thường.

## 5. Cần context theo request mà không muốn trả giá Request scope?

Dùng **AsyncLocalStorage** (Node built-in) — thường qua lib `nestjs-cls`: middleware/interceptor set context (tenant-id, request-id, user) vào ALS đầu request, service singleton **đọc từ ALS** thay vì nhận qua DI. Được cả hai: singleton performance + context per-request, và hoạt động cả trong async chain.

Trade-off: context "tàng hình" (không thấy trong signature hàm) → khó test hơn truyền tham số. Vì vậy thứ tự ưu tiên: **truyền tham số tường minh → ALS/CLS khi tham số phải xuyên quá nhiều tầng → Request scope là phương án cuối**.

## Tóm tắt phỏng vấn (30 giây)

> "Mặc định singleton cho tất cả — vừa nhanh vừa tương thích worker/cron/socket là những context không có request. Hợp đồng đi kèm là service stateless: state per-request nằm trên tham số hàm, không bao giờ trên field, vì instance dùng chung giữa các request song song — lưu nhầm là vừa leak vừa lẫn data chéo tenant. Request scope tôi tránh vì scope bubbling kéo cả cây DI ra khỏi cache và chết trong non-HTTP context; khi thật sự cần context xuyên tầng thì dùng AsyncLocalStorage/nestjs-cls thay vì phá scope. Leak trong singleton chủ yếu đến từ map in-memory không trần và listener đăng ký lặp — phòng bằng key hữu hạn/LRU và đăng ký một lần trong lifecycle hook."
