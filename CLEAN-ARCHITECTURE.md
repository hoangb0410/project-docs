# Clean Architecture

## 1. Định nghĩa

Clean Architecture là mô hình tổ chức mã nguồn do Robert C. Martin (Uncle Bob) đề xuất, trong đó **business logic nằm ở trung tâm hệ thống và không phụ thuộc vào bất kỳ chi tiết kỹ thuật nào bên ngoài** (framework, cơ sở dữ liệu, giao thức HTTP, thư viện). Mọi phụ thuộc (dependency) đều hướng từ lớp ngoài vào lớp trong.

Mục tiêu: business logic có thể được viết, kiểm thử và thay đổi độc lập với hạ tầng kỹ thuật; các chi tiết như DB hay framework chỉ là "plugin" có thể thay thế.

## 2. Các lớp (từ trong ra ngoài)

```
┌─────────────────────────────────────────────┐
│  Frameworks & Drivers                       │
│  (NestJS, Sequelize, PostgreSQL, SQS, ...)  │
│  ┌───────────────────────────────────────┐  │
│  │  Interface Adapters                   │  │
│  │  (Controllers, Presenters, Repos)     │  │
│  │  ┌─────────────────────────────────┐  │  │
│  │  │  Use Cases                      │  │  │
│  │  │  (Application Business Rules)   │  │  │
│  │  │  ┌───────────────────────────┐  │  │  │
│  │  │  │  Entities                 │  │  │  │
│  │  │  │  (Enterprise Rules)       │  │  │  │
│  │  │  └───────────────────────────┘  │  │  │
│  │  └─────────────────────────────────┘  │  │
│  └───────────────────────────────────────┘  │
└─────────────────────────────────────────────┘
        Dependency hướng từ ngoài vào trong →
```

### 2.1. Entities (Enterprise Business Rules)

- Chứa các quy tắc nghiệp vụ cốt lõi, thuần logic, ổn định nhất trong hệ thống.
- Không biết gì về DB, HTTP, framework.
- Ví dụ: quy tắc "một booking không được trùng khung giờ closeout của venue".

### 2.2. Use Cases (Application Business Rules)

- Điều phối các entity để thực hiện một nghiệp vụ cụ thể của ứng dụng.
- Định nghĩa **interface (port)** cho những gì nó cần từ bên ngoài (lưu trữ, gửi email, thanh toán) nhưng không biết implementation.
- Ví dụ: use case "tạo booking" gồm các bước kiểm tra pacing → phân bổ bàn → tính deposit.

### 2.3. Interface Adapters

- Lớp chuyển đổi dữ liệu giữa use case và thế giới bên ngoài.
- Gồm: controller (nhận request, gọi use case), presenter (định dạng response), repository implementation (hiện thực port lưu trữ bằng ORM cụ thể).

### 2.4. Frameworks & Drivers

- Chi tiết kỹ thuật ngoài cùng: web framework, ORM, message queue, dịch vụ bên thứ ba.
- Đây là lớp dễ thay thế nhất và ít quan trọng nhất đối với nghiệp vụ.

## 3. Dependency Rule — quy tắc quan trọng nhất

> Mã nguồn ở lớp trong không được import, tham chiếu hay biết đến bất kỳ thứ gì ở lớp ngoài.

Use case không biết nó đang chạy trên NestJS hay dữ liệu lưu ở PostgreSQL. Khi cần thao tác với bên ngoài, nó gọi qua **interface do chính nó định nghĩa**; lớp ngoài cung cấp implementation và được "tiêm" vào lúc runtime. Đây chính là nguyên lý **Dependency Inversion** (chữ D trong SOLID).

```
Controller ──▶ UseCase ──▶ IBookingRepository (interface, thuộc lớp trong)
                                   ▲
                                   │ implements
                     SequelizeBookingRepository (lớp ngoài)
```

## 4. Ví dụ cụ thể — feature "booking" theo Clean Architecture

### 4.1. Cấu trúc thư mục

```
src/modules/booking/
├── domain/                              # Lớp Entities — trong cùng
│   ├── booking.entity.ts                #   entity thuần, không dính ORM
│   └── booking.errors.ts                #   lỗi nghiệp vụ
│
├── application/                         # Lớp Use Cases
│   ├── create-booking.usecase.ts        #   nghiệp vụ "tạo booking"
│   ├── cancel-booking.usecase.ts
│   └── ports/
│       ├── booking-repository.port.ts   #   interface — use case ĐỊNH NGHĨA nhu cầu
│       └── notification.port.ts
│
├── infrastructure/                      # Lớp Frameworks & Drivers
│   ├── booking.model.ts                 #   Sequelize model — chỉ lớp này biết ORM
│   ├── sequelize-booking.repository.ts  #   implements BookingRepositoryPort
│   └── sendgrid-notification.adapter.ts #   implements NotificationPort
│
├── presentation/                        # Lớp Interface Adapters
│   ├── booking.controller.ts            #   HTTP in — gọi use case
│   └── dto/
│       ├── create-booking.dto.ts        #   shape của request
│       └── booking.response.ts          #   shape của response
│
└── booking.module.ts                    # wiring: bind port → adapter
```

### 4.2. Luồng một request đi qua các file

```
POST /bookings
   │
   ▼
booking.controller.ts        (presentation)
   │  validate DTO, KHÔNG có logic nghiệp vụ
   ▼
create-booking.usecase.ts    (application)
   │  logic nghiệp vụ: check overlap → tạo entity → lưu → notify
   │  chỉ biết booking-repository.port.ts (interface)
   ▼
booking-repository.port.ts   (application/ports — interface)
   ▲
   │  implements (được bind trong booking.module.ts)
   │
sequelize-booking.repository.ts  (infrastructure)
   │  duy nhất file này biết Sequelize
   ▼
booking.model.ts → PostgreSQL
```

### 4.3. Mã nguồn từng file

**`domain/booking.entity.ts`** — entity thuần, không import gì từ lớp ngoài:

```typescript
export class Booking {
  private constructor(
    public readonly venueId: number,
    public readonly partySize: number,
    public readonly startTime: Date,
    public status: BookingStatus,
  ) {}

  static create(input: { venueId: number; partySize: number; startTime: Date }): Booking {
    if (input.partySize < 1) throw new InvalidPartySizeError();
    return new Booking(input.venueId, input.partySize, input.startTime, BookingStatus.PENDING);
  }

  overlaps(other: Booking): boolean {
    return this.startTime.getTime() === other.startTime.getTime();
  }
}
```

**`application/ports/booking-repository.port.ts`** — use case định nghĩa nhu cầu, chưa có implementation:

```typescript
export interface BookingRepositoryPort {
  findByVenueAndTime(venueId: number, startTime: Date): Promise<Booking[]>;
  save(booking: Booking): Promise<Booking>;
}

export const BOOKING_REPOSITORY = Symbol('BOOKING_REPOSITORY');
```

**`application/create-booking.usecase.ts`** — toàn bộ logic nghiệp vụ, chỉ biết interface:

```typescript
@Injectable()
export class CreateBookingUseCase {
  constructor(
    @Inject(BOOKING_REPOSITORY) private readonly bookingRepo: BookingRepositoryPort,
    @Inject(NOTIFICATION_PORT) private readonly notifier: NotificationPort,
  ) {}

  async execute(input: CreateBookingInput): Promise<Booking> {
    const existing = await this.bookingRepo.findByVenueAndTime(input.venueId, input.startTime);
    const booking = Booking.create(input);
    if (existing.some((b) => booking.overlaps(b))) throw new SlotUnavailableError();

    const saved = await this.bookingRepo.save(booking);
    await this.notifier.sendConfirmation(saved);
    return saved;
  }
}
```

**`infrastructure/sequelize-booking.repository.ts`** — adapter, duy nhất nơi biết ORM:

```typescript
@Injectable()
export class SequelizeBookingRepository implements BookingRepositoryPort {
  constructor(@InjectModel(BookingModel) private readonly model: typeof BookingModel) {}

  async findByVenueAndTime(venueId: number, startTime: Date): Promise<Booking[]> {
    const rows = await this.model.findAll({ where: { venueId, startTime } });
    return rows.map(toDomainEntity); // map model → entity, không leak model ra ngoài
  }

  async save(booking: Booking): Promise<Booking> {
    const row = await this.model.create(toRow(booking));
    return toDomainEntity(row);
  }
}
```

**`presentation/booking.controller.ts`** — chỉ nhận request và gọi use case:

```typescript
@Controller('bookings')
export class BookingController {
  constructor(private readonly createBooking: CreateBookingUseCase) {}

  @Post()
  async create(@Body() dto: CreateBookingDto): Promise<BookingResponse> {
    const booking = await this.createBooking.execute(dto);
    return BookingResponse.from(booking);
  }
}
```

**`booking.module.ts`** — nơi duy nhất nối interface với implementation:

```typescript
@Module({
  controllers: [BookingController],
  providers: [
    CreateBookingUseCase,
    { provide: BOOKING_REPOSITORY, useClass: SequelizeBookingRepository },
    { provide: NOTIFICATION_PORT, useClass: SendGridNotificationAdapter },
  ],
})
export class BookingModule {}
```

### 4.4. Điểm mấu chốt

- Đổi `useClass: SequelizeBookingRepository` thành `DynamoBookingRepository` — use case và controller **không đổi một dòng nào**.
- Test `CreateBookingUseCase` chỉ cần mock 2 interface, không cần DB, không cần NestJS testing module nặng.
- So với nollie-api: luồng tương đương là `booking.controller.ts → booking.service.ts → booking.repository.ts → booking.model.ts` — cùng số bước, nhưng service inject thẳng concrete `BookingRepository` (không qua port/Symbol) và repository trả về Sequelize object thay vì domain entity. Đó chính là phần boilerplate mà nollie-api lược bỏ.

## 5. Lợi ích và cái giá

### Lợi ích

- **Testability**: kiểm thử business logic không cần DB, HTTP hay framework thật.
- **Độc lập framework**: đổi NestJS sang framework khác, hoặc REST sang GraphQL, không ảnh hưởng lớp trong.
- **Độc lập DB**: đổi PostgreSQL sang DynamoDB chỉ cần viết adapter mới.
- **Trì hoãn quyết định kỹ thuật**: có thể viết và kiểm thử nghiệp vụ trước khi chốt DB hay hạ tầng.

### Cái giá

- **Boilerplate lớn**: mỗi nghiệp vụ cần interface, DTO riêng cho từng lớp, mapper chuyển đổi giữa các lớp.
- **Overkill với CRUD đơn giản**: một endpoint đọc-ghi thẳng không hưởng lợi gì từ 4 lớp trừu tượng.
- **Đường cong học tập**: đội ngũ phải hiểu rõ dependency rule, nếu không sẽ tạo ra cấu trúc phức tạp nhưng vẫn phụ thuộc chéo.

## 6. So sánh với kiến trúc của nollie-api

nollie-api theo **layered architecture kiểu NestJS** — cùng tinh thần tách lớp nhưng thực dụng hơn:

| Khía cạnh | Clean Architecture | nollie-api |
|---|---|---|
| Tách controller / service / data | Có | Có (controller thin → service → repository) |
| Business logic tập trung một chỗ | Use case | `[feature].service.ts` |
| Truy cập dữ liệu qua lớp riêng | Repository qua interface | `*.repository.ts` extends `BaseRepository` |
| Dependency rule enforce tuyệt đối | Có — service chỉ biết interface | Không — service inject trực tiếp concrete repository class |
| Entity độc lập ORM | Có | Không — model gắn với Sequelize |

Đánh đổi của cách tiếp cận trong nollie-api: dependency rule không tuyệt đối, đổi lại ít boilerplate hơn đáng kể và phù hợp với dependency injection sẵn có của NestJS.

## 7. Khi nào nên áp dụng đầy đủ

- **Nên**: domain phức tạp, nghiệp vụ thay đổi thường xuyên, cần hỗ trợ nhiều loại hạ tầng (multi-DB, multi-transport), vòng đời dự án dài.
- **Không cần đầy đủ**: ứng dụng CRUD, dự án nhỏ, framework đã cung cấp cấu trúc lớp hợp lý (như NestJS) — khi đó áp dụng tinh thần (tách lớp, logic trong service, không truy vấn DB trong controller) là đủ.

## 8. Tài liệu tham khảo

- Robert C. Martin — *Clean Architecture: A Craftsman's Guide to Software Structure and Design* (2017)
- Bài blog gốc: [The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html) (2012)
- Các mô hình liên quan: Hexagonal Architecture (Ports & Adapters), Onion Architecture
