# NestJS Fundamental

## Request Lifecycle

### Mô hình tư duy

NestJS bọc route handler bằng nhiều lớp vỏ đồng tâm. Request đi vào từ lớp ngoài cùng vào tâm, response đi ra theo chiều ngược lại. Exception Filter là lưới hứng nằm ngay bên ngoài lớp vỏ do Nest quản lý.

```
                    ┌─────────────── Exception Filters ───────────────┐
                    │  ┌──────────── Global Guards ──────────────┐    │
Middleware ────────►│  │  ┌───── Global Interceptors (pre) ────┐ │    │
(ngoài phạm vi      │  │  │  ┌──── Controller Interceptors ──┐ │ │    │
 filter)            │  │  │  │  ┌── Route Interceptors ────┐ │ │ │    │
                    │  │  │  │  │  Pipes → Controller      │ │ │ │    │
                    │  │  │  │  │         → Service        │ │ │ │    │
                    │  │  │  │  └──────────────────────────┘ │ │ │    │
                    │  │  │  └───────────────────────────────┘ │ │    │
                    │  │  └── Interceptors (post) chạy ngược ──┘ │    │
                    │  └─────────────────────────────────────────┘    │
                    └─────────────────────────────────────────────────┘
```

Thứ tự đầy đủ và vai trò của từng tầng:

| #   | Stage                                | Thường dùng cho                                                                    |
| --- | ------------------------------------ | ---------------------------------------------------------------------------------- |
| 1   | Middleware                           | Xử lý tầng transport: parse raw request, security header, CORS, rate limit theo IP |
| 2   | Guards                               | **Authentication** (bạn là ai) và **Authorization** (bạn có quyền gì)              |
| 3   | Interceptors (pre)                   | Cross-cutting trước handler: tracing, đọc cache, bắt đầu đo latency                |
| 4   | Pipes                                | Làm sạch input: validation + transformation                                        |
| 5   | Controller                           | Định tuyến và định nghĩa HTTP contract                                             |
| 6   | Service                              | Business logic, database, external API                                             |
| 7   | Interceptors (post) — chạy **ngược** | Định hình output: response envelope, serialization, ghi cache                      |
| 8   | Exception Filters                    | Chuẩn hóa error response và quan sát lỗi (chỉ khi có exception ở bước 2–7)         |

Mỗi enhancer (Guard, Interceptor, Pipe, Filter) đều có 3 scope và chạy theo thứ tự: **global → controller → route** (Pipe có thêm cấp thứ tư: param). Riêng Exception Filter thì ngược lại: Nest chọn filter **cụ thể nhất** khớp với exception (route → controller → global) và chỉ **một** filter xử lý, không chain qua nhiều filter.

---

### 1. Middleware

**Chức năng:** Can thiệp trực tiếp vào cặp `req` / `res` ở tầng HTTP gốc (Express/Fastify), trước khi request bước vào enhancer pipeline của Nest. Dùng để biến đổi dữ liệu thô, thiết lập HTTP header, hoặc chặn kết nối bất hợp lệ từ tầng mạng.

**Thường dùng cho:** những việc **không cần biết route nào đang được gọi** — chuẩn hóa request thô (cookie, compression, body parser), bảo vệ tầng mạng (CORS, `helmet`, rate limit theo IP), access log thô, gán `request-id`, và giữ raw body để verify webhook signature.

- Ví dụ 1: Parse chuỗi Cookie Header từ request thành object `req.cookies`.
- Ví dụ 2: Cấu hình CORS để chỉ cho phép một số domain nhất định truy cập API.
- Ví dụ 3: Dùng `express-rate-limit` để giới hạn tối đa 100 request/phút cho mỗi địa chỉ IP.

**Lưu ý quan trọng:**

- Class middleware `@Injectable()` **vẫn nằm trong IoC container** và inject provider bình thường. Điểm khác biệt thật sự không phải là "chưa qua IoC", mà là middleware chỉ nhận `(req, res, next)` — **không có `ExecutionContext`**, nên không đọc được metadata của handler qua `Reflector` (không biết route nào, decorator nào sắp chạy).
- Middleware đăng ký bằng `app.use()` trong `main.ts` chạy **trước** middleware đăng ký qua `configure(consumer)` của module.
- **Exception Filter không bắt được lỗi throw từ middleware.** Lỗi ở đây rơi về error handler mặc định của Express/Fastify, không đi qua filter của Nest.
- Middleware chỉ tồn tại ở HTTP context. WebSocket gateway và microservice không có middleware (nhưng vẫn có Guard / Interceptor / Filter).

---

### 2. Guards

**Chức năng:** Quyết định request có được phép đi tiếp vào route handler hay không, dựa trên điều kiện logic (permission, role, ACL). Trả về boolean. Nếu `false`, Nest ngắt luồng ngay.

**Thường dùng cho:** **Authentication** (xác thực — request này là ai: JWT, session, API key) và **Authorization** (phân quyền — người đó được làm gì: role, permission, ACL). Ngoài ra: giới hạn phạm vi dữ liệu trong hệ multi-tenant (kiểu `VenueAccessGuard` — user có thuộc venue trong URL không), chặn route theo feature flag, và throttling ở mức route (`ThrottlerGuard`).

- Ví dụ 1: Kiểm tra JWT Token ở header để xác thực người dùng đã đăng nhập.
- Ví dụ 2: Kiểm tra user trong token có quyền Admin để vào route quản trị hay không.
- Ví dụ 3: Kiểm tra API Key ở header có hợp lệ với webhook từ bên thứ ba (Stripe, PayPal).

**Lưu ý quan trọng:**

- Guard trả `false` → Nest tự throw **`ForbiddenException` (403)**. Không phải 401. Muốn 401 thì phải **tự throw `UnauthorizedException`** trong guard (đây chính là điều `AuthGuard` của Passport làm).
- Guard là nơi đầu tiên có `ExecutionContext`, nên đọc được metadata qua `Reflector` — nền tảng cho pattern `@Roles('admin')` + `RolesGuard`.
- Guard chạy **trước Pipe**, nên `request.body` mà guard thấy là **raw body chưa validate, chưa transform**. Không dùng DTO đã ép kiểu bên trong guard.
- `@nestjs/throttler` (`ThrottlerGuard`) là **Guard**, không phải middleware — khác với `express-rate-limit` ở mục 1.
- Guard đăng ký bằng `app.useGlobalGuards()` **không inject được dependency**. Muốn inject thì đăng ký qua provider `APP_GUARD`.

---

### 3. Interceptors (pre-controller)

**Chức năng:** Áp dụng Aspect-Oriented Programming với RxJS để can thiệp vào luồng execution trước khi hàm ở Controller được gọi. Dùng để mở rộng logic, bổ sung dữ liệu vào request, hoặc hủy luồng bằng cách trả response sớm (response hijacking).

**Thường dùng cho:** logic cross-cutting cần **biết route nào đang chạy** nhưng không thuộc nghiệp vụ — distributed tracing / correlation ID, bắt đầu đo latency, đọc cache và trả sớm, audit log "ai gọi endpoint nào", idempotency check (key đã xử lý thì trả kết quả cũ), gán context (tenant, locale, timezone) vào request cho service dùng lại.

- Ví dụ 1: Ghi mốc thời gian bắt đầu nhận request để tính latency.
- Ví dụ 2: Kiểm tra cache; nếu có sẵn dữ liệu thì trả về ngay cho client, không chạy vào Controller/Service.
- Ví dụ 3: Gắn thông tin ngữ cảnh (Trace ID) vào request để phục vụ distributed tracing.

**Lưu ý quan trọng:**

- "Hủy luồng" nghĩa là **không gọi `next.handle()`** mà `return of(cachedValue)`. Chỉ cần gọi `next.handle()` là Controller chắc chắn chạy.
- Giống Guard: interceptor pre-phase chạy **trước Pipe**, nên chỉ thấy raw body.
- Phần code trước `next.handle()` là pre-phase; phần trong `.pipe(map/tap/catchError)` là post-phase (mục 7). Cùng một class, hai thời điểm.

---

### 4. Pipes

**Chức năng:** Xử lý dữ liệu đầu vào của các tham số (`@Body()`, `@Query()`, `@Param()`). Hai nhiệm vụ chính: **Transformation** (ép kiểu dữ liệu thô về kiểu mong muốn) và **Validation** (đánh giá theo rule, throw `BadRequestException` 400 nếu sai).

**Thường dùng cho:** bảo đảm business logic **không bao giờ nhận dữ liệu bẩn** — validate DTO theo rule, ép kiểu từ query string (`string` → `number` / `boolean` / `Date`), gán default cho `page` / `limit`, loại bỏ field lạ (`whitelist: true`), và parse các định dạng đặc thù (enum, UUID, khoảng ngày).

- Ví dụ 1: Chuyển tham số `id` từ string sang number (`ParseIntPipe`).
- Ví dụ 2: Validate DTO bằng `class-validator` (báo lỗi nếu email sai định dạng hoặc password quá ngắn).
- Ví dụ 3: Gán giá trị mặc định cho `page` và `limit` nếu client không truyền (`DefaultValuePipe`).

**Lưu ý quan trọng:**

- Pipe chạy **theo từng parameter**, ngay tại thời điểm Nest resolve argument cho handler. Param nào không có decorator thì không đi qua pipe nào.
- `ValidationPipe` chỉ transform sang instance của class DTO khi bật `transform: true`; không bật thì `@Body()` vẫn là plain object và `class-transformer` decorator không có tác dụng.
- Đây là điểm cuối cùng còn can thiệp được vào input trước khi vào business logic. Sau bước này, dữ liệu coi như đã sạch.

---

### 5. Controller

**Chức năng:** Lớp giao tiếp chính của API (API endpoint layer). Map HTTP method (GET, POST, PUT, DELETE) với đường dẫn URL, tiếp nhận tham số đã qua Pipe và gọi phương thức tương ứng ở tầng Service.

**Thường dùng cho:** khai báo **HTTP contract** của API — route path và method, gắn enhancer theo route (`@UseGuards`, `@UseInterceptors`), set status code / header (`@HttpCode`, `@Header`), tài liệu Swagger (`@ApiOperation`, `@ApiResponse`), và chọn service phù hợp để gọi.

- Ví dụ 1: Nhận `POST /users` và chuyển DTO sang `UserService`.
- Ví dụ 2: Nhận `GET /products?category=shoes` và gọi `ProductService.findByCategory('shoes')`.
- Ví dụ 3: Định tuyến `DELETE /posts/:id` và truyền `id` sang `PostService.delete()`.

**Lưu ý:** Controller nên mỏng, không chứa business logic, không truy vấn DB trực tiếp.

---

### 6. Service

**Chức năng:** Tầng chứa toàn bộ nghiệp vụ cốt lõi (core business logic). Xử lý thuật toán, quản lý trạng thái, thao tác CRUD với database qua ORM/Repository, điều phối dữ liệu với external service.

**Thường dùng cho:** mọi thứ mang tính **nghiệp vụ và tái sử dụng được** — quy tắc domain, transaction nhiều bảng, gọi API bên ngoài, đẩy job vào queue, phát event / socket, và orchestration giữa nhiều repository. Đây cũng là tầng duy nhất nên viết unit test cho logic, vì nó không phụ thuộc HTTP.

- Ví dụ 1: Tương tác database để tạo người dùng mới, mã hóa mật khẩu bằng bcrypt.
- Ví dụ 2: Gọi API bên thứ ba (VNPay) để tạo giao dịch thanh toán.
- Ví dụ 3: Tính tổng tiền hóa đơn, áp mã giảm giá, trừ số lượng hàng tồn kho.

**Lưu ý:** Service không phải một "stage" của lifecycle theo nghĩa enhancer — nó là provider được Controller gọi. Nhưng đặt vào sơ đồ để thấy rõ đâu là tâm của luồng thì hợp lý.

---

### 7. Interceptors (post-controller)

**Chức năng:** Biến đổi hoặc thao tác trên giá trị trả về (Observable stream) từ Controller, trước khi dữ liệu được chuyển thành HTTP response. Dùng để chuẩn hóa response format, ghi cache, đo thời gian xử lý toàn luồng.

**Thường dùng cho:** định hình **output** và các side-effect sau khi đã có kết quả — bọc response envelope dùng chung, serialization / ẩn field nhạy cảm, chuẩn hóa cấu trúc pagination, ghi cache, áp `timeout`, log latency và kích thước payload.

- Ví dụ 1: Chuẩn hóa response thành `{ statusCode: 200, data: result, message: "Success" }`.
- Ví dụ 2: Loại bỏ trường nhạy cảm (`password`, `refreshToken`) khỏi object trả về.
- Ví dụ 3: Lưu kết quả vừa query từ database vào Redis cache cho request sau dùng lại.

**Lưu ý quan trọng:**

- Post-phase chạy **theo thứ tự ngược** với pre-phase: route-scoped xong trước, controller-scoped, rồi global-scoped xong cuối cùng. Vì các interceptor lồng nhau, global là lớp ngoài nhất.
- Hệ quả thực tế: interceptor chuẩn hóa response nên đặt ở **global**, để nó bọc kết quả cuối cùng sau khi mọi interceptor bên trong đã xử lý xong.
- Interceptor cũng bắt được lỗi qua `catchError` — nên nếu muốn ghi log lỗi kèm latency, làm ở đây, không nhất thiết ở filter.
- `ClassSerializerInterceptor` (`@Exclude()`) chính là ví dụ 2 dưới dạng built-in.

---

### 8. Exception Filters

**Chức năng:** Bẫy các exception chưa được xử lý phát sinh trong vòng đời request. Cho phép thay đổi HTTP status code, log lỗi chi tiết, và kiểm soát cấu trúc JSON error response trả về client.

**Thường dùng cho:** giữ **một error contract duy nhất** cho toàn bộ API và quan sát lỗi tập trung — map lỗi hạ tầng (Sequelize, Stripe, HTTP client) sang status code đúng, dịch message lỗi qua i18n, che chi tiết nội bộ (stack trace, tên bảng) khỏi client, và đẩy alert sang Sentry / Slack.

- Ví dụ 1: Chuyển lỗi crash không mong muốn thành `{ statusCode: 500, message: "Internal server error" }`.
- Ví dụ 2: Bắt lỗi database (trùng unique key) và chuyển thành 409 Conflict thân thiện hơn.
- Ví dụ 3: Đẩy thông báo lỗi nghiêm trọng về Slack hoặc Sentry.

**Lưu ý quan trọng:**

- Phạm vi phủ: **Guard → Interceptor → Pipe → Controller → Service**. **Không** phủ Middleware.
- Nest chọn **một** filter cụ thể nhất khớp exception type (route → controller → global) và dừng ở đó. Không có chuỗi filter nối tiếp nhau.
- Filter chỉ hoạt động khi response **chưa được gửi**. Lỗi phát sinh sau khi stream đã bắt đầu ghi (ví dụ streaming file) thì không cứu được.
- Filter đăng ký bằng `app.useGlobalFilters()` không inject được dependency; muốn inject thì dùng provider `APP_FILTER`.

---

### Những gotcha đáng nhớ

| Vấn đề                                                      | Nguyên nhân                                                   |
| ----------------------------------------------------------- | ------------------------------------------------------------- |
| Guard đọc `request.body` thấy kiểu sai / thiếu default      | Guard chạy trước Pipe, body còn raw                           |
| Middleware throw lỗi nhưng response không đúng format chuẩn | Filter không phủ middleware                                   |
| Guard `false` mà nhận 403 trong khi muốn 401                | Mặc định của Nest là `ForbiddenException`                     |
| Interceptor chuẩn hóa response bị interceptor khác ghi đè   | Post-phase chạy ngược; global bọc ngoài cùng                  |
| Global guard/filter inject `undefined`                      | Đăng ký qua `useGlobalX()` thay vì `APP_GUARD` / `APP_FILTER` |
| DTO không được transform dù có decorator                    | `ValidationPipe` chưa bật `transform: true`                   |
| `ParseIntPipe` không chạy                                   | Param không có decorator (`@Param('id')`)                     |

---

## Dependency Injection

### Mô hình tư duy

Class không tự tạo ra thứ nó cần. Nó chỉ **khai báo** ở constructor rằng "tôi cần cái này", còn IoC Container của Nest tra bảng đăng ký (provider), tạo hoặc lấy lại instance, rồi đưa vào. Hệ quả: class phụ thuộc vào **kiểu** của dependency, không phụ thuộc vào **cách khởi tạo** nó.

### Không có DI so với có DI

```typescript
// ❌ Tight coupling — tự new bên trong
class BookingService {
  private mailer = new SendGridMailer(process.env.SENDGRID_KEY);
  private repo = new BookingRepository(new Sequelize(config));

  async confirm(id: number) {
    await this.repo.update(id, { status: "CONFIRMED" });
    await this.mailer.send(/* ... */);
  }
}
```

Vấn đề: `BookingService` biết quá nhiều — biết mailer là SendGrid, biết cách lấy API key, biết cách tạo kết nối DB. Muốn test `confirm()` là phải có key thật và DB thật. Muốn đổi sang provider mail khác là phải sửa chính file này.

```typescript
// ✅ DI — chỉ khai báo nhu cầu
@Injectable()
export class BookingService {
  constructor(
    private readonly mailer: MailerService,
    private readonly repo: BookingRepository,
  ) {}

  async confirm(id: number) {
    await this.repo.update(id, { status: "CONFIRMED" });
    await this.mailer.send(/* ... */);
  }
}
```

`BookingService` giờ chỉ quan tâm "có một `MailerService` để gọi `send()`". Ai tạo nó, cấu hình thế nào, là chuyện của module.

### Vai trò

| Vai trò                            | Ý nghĩa thực tế                                                                                                                                                |
| ---------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Tránh phụ thuộc chặt**           | Class phụ thuộc vào abstraction, không vào implementation cụ thể. Đổi `SendGridMailer` sang `SesMailer` chỉ cần sửa provider trong module, không sửa consumer. |
| **Dễ mock khi test**               | Constructor là điểm duy nhất để thay dependency, nên test chỉ cần truyền object giả — không cần DB, không cần network.                                         |
| **Quản lý lifecycle**              | Container giữ singleton, nên connection pool, Redis client, HTTP agent được chia sẻ thay vì tạo lại mỗi lần dùng.                                              |
| **Khai báo dependency tường minh** | Nhìn constructor là biết class này chạm vào những gì. Blast radius của một thay đổi trở nên đọc được, và vòng phụ thuộc lộ ra sớm.                             |
| **Tách cấu hình khỏi logic**       | Đọc env, ghép URL, chọn region nằm ở factory provider; service chỉ nhận kết quả đã sẵn sàng.                                                                   |
| **Tái sử dụng thật sự**            | Một provider dùng được ở HTTP request, queue worker, cron job — vì nó không gắn với ngữ cảnh nào cả.                                                           |

### Ba mảnh ghép

1. **Provider** — công thức để container biết tạo ra cái gì. Khai báo trong `providers` của module.
2. **Token** — khóa tra cứu. Thường chính là class (`MailerService`), hoặc string/symbol khi dependency không phải class.
3. **Injector** — bộ máy đọc metadata constructor, resolve từng token, và cache instance lại.

`@Injectable()` chỉ có một nhiệm vụ: cho phép TypeScript phát metadata kiểu tham số constructor để injector đọc được. Class **không có dependency nào** thì về lý thuyết không cần decorator này, nhưng vẫn nên gắn cho nhất quán.

### Các kiểu provider

```typescript
@Module({
  providers: [
    // 1. useClass (dạng viết tắt) — phổ biến nhất
    BookingService,

    // 2. useClass với token khác — đổi implementation không sửa consumer
    { provide: MailerService, useClass: process.env.REGION === 'uk' ? SesMailer : SendGridMailer },

    // 3. useValue — inject config, constant, hoặc mock
    { provide: 'STRIPE_CONFIG', useValue: { apiVersion: '2024-06-20' } },

    // 4. useFactory — cần tính toán hoặc cần dependency khác để khởi tạo
    {
      provide: 'REDIS_CLIENT',
      useFactory: (config: ConfigService) => new Redis(config.get('REDIS_URL')),
      inject: [ConfigService],
    },

    // 5. useExisting — tạo alias cho provider đã có
    { provide: 'LegacyMailer', useExisting: MailerService },
  ],
})
```

Với token không phải class, chỗ nhận phải chỉ rõ token:

```typescript
constructor(@Inject('REDIS_CLIENT') private readonly redis: Redis) {}
```

**Lưu ý:** không dùng `interface` làm token được. Interface bị xóa khi compile sang JavaScript nên không còn gì để tra cứu — muốn inject theo abstraction thì dùng abstract class hoặc string/symbol token.

### Phạm vi module

- `providers` — những gì module này tạo ra và dùng nội bộ.
- `exports` — tập con của `providers` cho phép module khác dùng.
- `imports` — mượn provider đã export của module khác.

Provider không nằm trong `exports` thì module khác **không thấy**, dù đã `imports`. Ngược lại, nhiều module cùng import một module thì vẫn **dùng chung một instance** — import không tạo bản sao.

### Injection scope

| Scope       | Hành vi                                                                                                         |
| ----------- | --------------------------------------------------------------------------------------------------------------- |
| `DEFAULT`   | Singleton toàn ứng dụng. Mặc định, và gần như luôn là lựa chọn đúng.                                            |
| `REQUEST`   | Một instance mới cho mỗi request. Dùng khi cần state riêng theo request (tenant context, transaction hiện tại). |
| `TRANSIENT` | Instance riêng cho mỗi chỗ inject. Hiếm dùng.                                                                   |

Cảnh báo về `REQUEST` scope: nó **lan lên toàn bộ chain**. Nếu một repository là request-scoped, mọi service dùng nó cũng trở thành request-scoped, và controller cũng vậy — Nest phải dựng lại cả nhánh cây phụ thuộc cho từng request. Chi phí này thật, nên cân nhắc truyền context qua tham số hàm trước khi chọn `REQUEST`.

### Circular dependency

Hai module hoặc hai service cần nhau sẽ làm injector bí, vì không biết tạo cái nào trước. `forwardRef()` là cách vá:

```typescript
constructor(
  @Inject(forwardRef(() => BookingService))
  private readonly bookingService: BookingService,
) {}
```

Nhưng đây là dấu hiệu thiết kế cần xem lại. Hướng bền hơn: **tách phần dùng chung ra một provider lá** (repository, hoặc một service không phụ thuộc ai) rồi cho cả hai bên dùng nó. Lưu ý `forwardRef()` chỉ xử lý vòng lặp ở tầng **injector** — vòng lặp ở tầng `import` giữa các file vẫn có thể làm module không boot được, và trường hợp đó phải cắt import thật sự.

### Vì sao DI làm test dễ hơn

Vì constructor là điểm duy nhất dependency đi vào, test chỉ cần đưa object giả vào đúng token đó:

```typescript
const mockRepo = { update: jest.fn(), findByPk: jest.fn() };
const mockMailer = { send: jest.fn() };

const moduleRef = await Test.createTestingModule({
  providers: [
    BookingService,
    { provide: BookingRepository, useValue: mockRepo },
    { provide: MailerService, useValue: mockMailer },
  ],
}).compile();

const service = moduleRef.get(BookingService);

it("gửi mail sau khi confirm", async () => {
  await service.confirm(1);
  expect(mockRepo.update).toHaveBeenCalledWith(1, { status: "CONFIRMED" });
  expect(mockMailer.send).toHaveBeenCalled();
});
```

Không DB, không SendGrid, không network. Nếu `BookingService` tự `new` dependency như ví dụ đầu, đoạn test này không viết được — đó chính là lý do "khó test" thường là triệu chứng của coupling, không phải của thiếu test framework.

### Những gotcha đáng nhớ

| Vấn đề                                                    | Nguyên nhân                                                                                           |
| --------------------------------------------------------- | ----------------------------------------------------------------------------------------------------- |
| `Nest can't resolve dependencies of X`                    | Provider chưa khai báo trong `providers`, hoặc module chứa nó chưa `exports`                          |
| Inject theo interface nhưng nhận `undefined`              | Interface bị erase khi compile — phải dùng token cụ thể qua `@Inject()`                               |
| Global guard/interceptor/filter inject `undefined`        | Đăng ký bằng `useGlobalX()` nằm ngoài container — dùng `APP_GUARD` / `APP_INTERCEPTOR` / `APP_FILTER` |
| API chậm bất thường sau khi thêm một provider             | Provider đó là `REQUEST` scope và đã kéo cả chain thành request-scoped                                |
| `forwardRef` rồi vẫn crash lúc boot                       | Vòng lặp ở tầng file import, không phải tầng injector                                                 |
| Hai module tưởng có instance riêng                        | Provider là singleton — import không nhân bản, muốn riêng phải khai báo provider riêng                |
| Config đọc được ở service này, `undefined` ở service khác | `ConfigModule` chưa `isGlobal: true` và module kia chưa import                                        |

---

## Design Patterns trong NestJS

### Mô hình tư duy

NestJS không phải framework "tình cờ" dùng vài pattern. Gần như mọi cơ chế nhìn thấy được của nó là một pattern kinh điển được đặt tên lại: `useFactory` là Factory, `PassportStrategy` là Strategy, chuỗi middleware → guard → interceptor là Chain of Responsibility. Nhận ra pattern đằng sau giúp đoán được API mà không cần tra docs.

### Bản đồ nhanh

| Nhóm       | Pattern                 | Dẫn chứng trong Nest                                |
| ---------- | ----------------------- | --------------------------------------------------- |
| Creational | Factory                 | `useFactory`, `NestFactory.create()`                |
| Creational | Singleton               | `Scope.DEFAULT` — provider mặc định                 |
| Creational | Builder                 | `new DocumentBuilder().setTitle().build()`          |
| Creational | Prototype               | `Scope.TRANSIENT`                                   |
| Structural | Adapter                 | `ExpressAdapter` / `FastifyAdapter`, `IoAdapter`    |
| Structural | Facade                  | Service bọc repository + SDK ngoài, `ConfigService` |
| Structural | Proxy                   | Router proxy bọc handler, `ModuleRef` lazy resolve  |
| Structural | Composite               | Cây module (`imports` lồng nhau)                    |
| Structural | Decorator (GoF)         | Interceptor bọc `next.handle()`                     |
| Behavioral | Chain of Responsibility | Middleware → Guard → Interceptor → Pipe             |
| Behavioral | Strategy                | `PipeTransform`, `PassportStrategy`, cache store    |
| Behavioral | Observer                | `Observable` trong interceptor, `@OnEvent`          |
| Behavioral | Template Method         | `BaseExceptionFilter`, abstract `validate()`        |
| Behavioral | Command                 | `@nestjs/cqrs` command + handler, queue job         |
| Behavioral | Mediator                | `CommandBus` / `EventBus` / `QueryBus`              |

---

### Creational — tạo ra object

Nhóm pattern trả lời câu hỏi **"object này được sinh ra như thế nào"**. Mục tiêu chung: chỗ _dùng_ object không phải chịu trách nhiệm _tạo_ object. Nhờ vậy khi cách tạo thay đổi — đổi implementation, thêm config, chuyển từ mỗi-lần-một-bản sang dùng chung — code đang dùng không phải sửa theo.

**Factory**

- _Lý thuyết:_ Giao việc khởi tạo cho một chỗ chuyên trách. Bên gọi chỉ nói "cho tôi loại này", không tự `new`, nên toàn bộ logic chọn implementation nằm gọn tại một điểm.
- _Trong Nest:_ provider `useFactory` — cần tính toán hoặc cần dependency khác mới tạo được object.

```typescript
{
  provide: 'REDIS_CLIENT',
  useFactory: (config: ConfigService) => new Redis(config.get('REDIS_URL')),
  inject: [ConfigService],
}
```

Cũng thấy ở `NestFactory.create(AppModule)` (dựng cả application từ metadata module) và ở pattern chọn implementation theo runtime kiểu `POSServiceFactory.create(provider)`.

**Singleton**

- _Lý thuyết:_ Bảo đảm một class chỉ tồn tại đúng một instance trong suốt vòng đời ứng dụng, và mọi nơi đều dùng chung instance đó.
- _Trong Nest:_ `Scope.DEFAULT` — scope mặc định của provider, nên `ConfigService`, connection pool, Redis client chỉ có một bản.

Khác biệt đáng nhớ: đây là **container-managed singleton**, không phải Singleton kinh điển kiểu `getInstance()` với biến static. Vòng đời do container giữ, class vẫn `new` được bình thường trong test — nên nó có lợi ích của Singleton mà không có nhược điểm không-test-được.

**Builder**

- _Lý thuyết:_ Dựng object nhiều tham số qua từng bước có tên rõ ràng thay vì một constructor dài khó đọc, và chốt lại bằng một lệnh `build()`.
- _Trong Nest:_ `DocumentBuilder` của Swagger.

```typescript
const config = new DocumentBuilder()
  .setTitle("Nollie API")
  .setVersion("1.0")
  .addBearerAuth()
  .build();
```

`Test.createTestingModule({...}).overrideProvider(X).useValue(mock).compile()` cũng là builder dạng fluent.

**Prototype**

- _Lý thuyết:_ Thay vì chia sẻ một object dùng chung, mỗi nơi nhận một bản riêng — để tự do thay đổi state cục bộ mà không ảnh hưởng nơi khác.
- _Trong Nest:_ `Scope.TRANSIENT`, dùng khi provider giữ state không được lẫn giữa các consumer (ví dụ logger mang tên class của chỗ inject nó).

---

### Structural — ghép các object lại

Nhóm pattern trả lời câu hỏi **"các object ghép với nhau ra sao"**. Mục tiêu chung: dựng được cấu trúc lớn từ nhiều thành phần rời mà vẫn giữ chúng thay thế được — thường bằng cách đặt một lớp trung gian vào giữa để **bọc**, **che**, hoặc **chuyển đổi** interface.

**Adapter**

- _Lý thuyết:_ Bọc một interface không tương thích thành interface mà bên gọi mong đợi — giống đầu chuyển đổi phích cắm. Hai bên không cần biết nhau, chỉ cần biết adapter.
- _Trong Nest:_ pattern rõ nhất của framework — core không biết gì về Express hay Fastify, nó chỉ nói chuyện với `AbstractHttpAdapter`.

```typescript
NestFactory.create<NestFastifyApplication>(AppModule, new FastifyAdapter());
```

Đổi một dòng này là đổi toàn bộ HTTP engine mà controller không cần sửa. Tương tự ở WebSocket (`IoAdapter` cho socket.io, `WsAdapter` cho ws) và ở microservice transport (Redis, NATS, SQS).

**Facade**

- _Lý thuyết:_ Cung cấp một cửa vào đơn giản cho một hệ thống con phức tạp, giấu bớt số bước và số thành phần bên trong. Bên gọi làm một lời gọi thay vì điều phối năm object.
- _Trong Nest:_ tầng Service — controller gọi `bookingService.confirm(id)` mà không cần biết bên trong có transaction, hai repository, một lần gọi Stripe và một job đẩy vào queue. `ConfigService` là facade cho `process.env` + validation + default.

**Proxy**

- _Lý thuyết:_ Một object mang cùng interface với object thật, nhận lời gọi trước để chèn thêm việc (kiểm tra quyền, cache, lazy load, bọc lỗi) rồi mới chuyển vào bên trong.
- _Trong Nest:_ mỗi route handler được bọc trong một router proxy để thiết lập "exception zone" — đó là lý do exception ném ra trong handler tới được exception filter, còn exception ném ra trong middleware (nằm ngoài proxy) thì không. `ModuleRef.get()` cũng cho phép lấy dependency trễ thay vì inject trực tiếp.

**Composite**

- _Lý thuyết:_ Cho phép đối xử với một object đơn lẻ và một nhóm object lồng nhau theo cùng một interface, nên code duyệt cây không cần phân biệt lá hay nhánh.
- _Trong Nest:_ cây module — một module import module khác, module đó lại import module khác nữa, nhưng ở mọi tầng nó vẫn chỉ là "một module" với cùng bộ metadata `imports / providers / exports`.

**Decorator (GoF)**

- _Lý thuyết:_ Bọc object gốc trong một lớp cùng interface để thêm hành vi trước và sau, xếp được nhiều lớp lên nhau. Khác kế thừa ở chỗ nó lắp vào lúc runtime, không cố định lúc compile.
- _Trong Nest:_ **Interceptor**.

```typescript
intercept(ctx: ExecutionContext, next: CallHandler) {
  const start = Date.now();
  return next.handle().pipe(tap(() => this.logger.log(`${Date.now() - start}ms`)));
}
```

`next.handle()` là hành vi gốc; phần trước và sau là lớp bọc. Nhiều interceptor lồng nhau chính là nhiều lớp decorator xếp lên nhau.

---

### Behavioral — điều phối luồng và giao tiếp

Nhóm pattern trả lời câu hỏi **"các object nói chuyện và chia việc với nhau thế nào"**. Mục tiêu chung: phân bổ trách nhiệm và điều phối luồng chạy sao cho thêm hoặc bớt một bước không buộc phải sửa những bước còn lại.

**Chain of Responsibility**

- _Lý thuyết:_ Xếp các handler thành một chuỗi; mỗi handler tự quyết định xử lý, chuyển tiếp, hay chặn lại. Bên gửi không biết ai sẽ xử lý cuối cùng, nên chèn thêm một mắt vào giữa chuỗi là đủ.
- _Trong Nest:_ toàn bộ request lifecycle ở section trên — middleware gọi `next()`, interceptor gọi `next.handle()`, guard trả `false` để cắt chuỗi.

**Strategy**

- _Lý thuyết:_ Tách một cách làm (thuật toán) ra thành object riêng, các cách làm khác nhau cùng chung interface, để chọn hoặc đổi lúc runtime mà không sửa bên gọi.
- _Trong Nest:_ Passport gọi thẳng pattern này bằng tên.

```typescript
export class JwtStrategy extends PassportStrategy(Strategy, "jwt") {
  /* ... */
}
```

Ngoài ra: mọi Pipe đều implement `PipeTransform` nên `ParseIntPipe` và `ValidationPipe` lắp vào cùng một chỗ; cache store (memory / Redis) và POS provider cũng là strategy.

**Observer**

- _Lý thuyết:_ Một bên phát tín hiệu, nhiều bên đăng ký nghe. Bên phát không biết ai đang nghe, nên thêm listener mới không cần sửa bên phát.
- _Trong Nest:_ RxJS `Observable` là kiểu trả về của interceptor, nên `map`, `tap`, `catchError`, `timeout` áp được lên stream response. Ở tầng ứng dụng thì là `EventEmitterModule`:

```typescript
this.eventEmitter.emit('booking.confirmed', { bookingId });

@OnEvent('booking.confirmed')
handleConfirmed(payload: BookingConfirmedEvent) { /* ... */ }
```

Thêm một listener mới không cần sửa chỗ `emit`.

**Template Method**

- _Lý thuyết:_ Class cha giữ bộ khung của thuật toán và thứ tự các bước, chỉ để lại vài bước cho class con override. Trật tự chung được bảo toàn, phần khác biệt được khu trú.
- _Trong Nest:_ `BaseExceptionFilter`.

```typescript
@Catch()
export class AllExceptionsFilter extends BaseExceptionFilter {
  catch(exception: unknown, host: ArgumentsHost) {
    this.logger.error(exception);
    super.catch(exception, host); // khung xử lý mặc định vẫn giữ
  }
}
```

Cũng thấy ở `PassportStrategy` (bắt buộc override `validate()`) và ở các `BaseRepository` tự viết: khung CRUD ở lớp cha, repository con chỉ khai báo model.

**Command**

- _Lý thuyết:_ Đóng gói một yêu cầu (hành động + dữ liệu) thành object. Vì yêu cầu giờ là dữ liệu, nó truyền đi được, xếp hàng được, log và retry được.
- _Trong Nest:_ rõ nhất ở `@nestjs/cqrs`.

```typescript
class ConfirmBookingCommand {
  constructor(public readonly bookingId: number) {}
}

@CommandHandler(ConfirmBookingCommand)
export class ConfirmBookingHandler implements ICommandHandler<ConfirmBookingCommand> {
  /* ... */
}
```

Mọi queue job (SQS/BullMQ) cũng là Command: payload là yêu cầu được serialize để một process khác thực thi sau.

**Mediator**

- _Lý thuyết:_ Các thành phần không gọi trực tiếp nhau mà gửi qua một trung gian, biến quan hệ nhiều-nhiều thành nhiều-một. Mỗi bên chỉ cần biết trung gian.
- _Trong Nest:_ `CommandBus`, `QueryBus`, `EventBus` của `@nestjs/cqrs` — bên gửi chỉ biết bus, không biết handler nào sẽ nhận. Đây là cách thay thế việc inject chéo service giữa các module, và cũng là cách tránh circular dependency đã nói ở section DI.

---

### Những chỗ dễ nhầm

| Nhầm lẫn                                       | Thực tế                                                                                                                 |
| ---------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| `@Injectable()`, `@Get()` là Decorator pattern | Đây là **decorator của TypeScript** — chỉ gắn metadata, không bọc hành vi. GoF Decorator trong Nest là Interceptor.     |
| Dependency Injection là một design pattern GoF | DI là **nguyên lý** (Inversion of Control), không nằm trong 23 pattern GoF. Nó là nền để các pattern trên lắp vào được. |
| Repository / Module / DTO là GoF pattern       | Đây là pattern **kiến trúc** (từ Fowler, DDD), không phải GoF.                                                          |
| Middleware là Decorator                        | Là Chain of Responsibility — nó chuyển tiếp qua `next()`, không bọc và biến đổi kết quả.                                |
| Singleton trong Nest là Singleton kinh điển    | Vòng đời do container quản lý, không dùng static `getInstance()` — nên vẫn test được bình thường.                       |
