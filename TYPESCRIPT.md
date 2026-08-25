# TypeScript Notes

## 1. TypeScript vs JavaScript

### 1.1. Kiểu chặt — điểm hơn cốt lõi

TypeScript có hệ thống kiểu chặt (static + strict typing), còn JavaScript là kiểu động và lỏng.

Trong JS:

```javascript
let x = 1;
x = "hello";        // đổi kiểu tùy ý, không ai chặn
"1" + 1;            // "11" — tự ép kiểu ngầm (weak typing)
user.emial;         // gõ sai tên field → undefined, chạy mới lỗi
```

Trong TS:

```typescript
let x: number = 1;
x = "hello";        // ❌ compile error: Type 'string' is not assignable to 'number'
user.emial;         // ❌ compile error: Property 'emial' does not exist
```

Kiểu chặt nghĩa là:

- **Biến/tham số/giá trị trả về có kiểu cố định** — gán sai kiểu, truyền thiếu field, gọi thuộc tính không tồn tại đều bị compiler chặn ngay, không đợi runtime.
- **Không ép kiểu ngầm bừa bãi** — muốn đổi kiểu phải nói rõ (cast, chuyển đổi tường minh).
- **Bật `strict` mode** thì chặt thêm: không cho `any` ngầm định (`noImplicitAny`), bắt xử lý `null`/`undefined` trước khi dùng (`strictNullChecks`).

Mọi lợi ích khác (autocomplete chính xác, refactor an toàn, interface làm contract) đều là hệ quả của việc compiler biết kiểu của mọi thứ.

### 1.2. Compile sang JavaScript — bằng gì, như thế nào

Node và browser chỉ chạy được JS, nên TS **bắt buộc phải compile sang JS** trước khi chạy.

**Bằng gì:** compiler chính thức là **`tsc`** (package `typescript`), cấu hình qua `tsconfig.json`.

**Quá trình `tsc` làm gì:**

1. **Parse** — đọc file `.ts` thành cây cú pháp (AST).
2. **Type-check** — kiểm tra toàn bộ kiểu; sai thì báo lỗi và dừng.
3. **Emit** — phát ra file `.js`:
   - **Xóa toàn bộ type** (type erasure): annotation, interface, generic biến mất hoàn toàn — file JS đầu ra không còn dấu vết TS.
   - **Hạ cú pháp (downlevel)** về phiên bản JS trong `tsconfig.target` (ví dụ `target: "ES2020"` thì cú pháp mới hơn được viết lại tương đương).
   - Chuyển hệ module theo `tsconfig.module` (ESM → CommonJS chẳng hạn).

```
main.ts ──parse──▶ AST ──type-check──▶ OK ──emit──▶ main.js (không còn type)
                                    └─▶ lỗi kiểu → báo lỗi, không emit
```

**Các công cụ compile khác:** esbuild, swc, Babel, `ts-node` — chỉ làm bước **xóa type + hạ cú pháp**, nhanh hơn nhiều nhưng **không type-check**. Vì vậy dự án thường chạy song song: swc/esbuild để build/chạy dev, và `tsc --noEmit` (chỉ type-check, không phát file) trong CI.

**Hệ quả của type erasure:** type chỉ tồn tại lúc compile, không kiểm tra được dữ liệu runtime (payload từ request chẳng hạn) — cần validation thật như `class-validator`.

### 1.3. Hai hiểu nhầm thường gặp

**"TS là ngôn ngữ biên dịch, JS là ngôn ngữ thông dịch?"** — Không hẳn. Đây là hai tầng khác nhau:

- Bước compile của TS là **source-to-source** (transpile): TS → JS, xảy ra **trước khi chạy**, xong là hết vai trò.
- Còn JS được thực thi thế nào là chuyện của **engine** (V8, JavaScriptCore...): engine hiện đại không thông dịch thuần mà dùng **thông dịch + JIT compile** — code chạy nhiều lần được compile xuống mã máy ngay lúc chạy.
- Nghĩa là: TS compile sang JS, rồi JS đó chạy y hệt mọi JS khác. TS không thay đổi cách code được thực thi.

**"TS chạy chậm hơn JS vì thêm bước convert?"** — Sai. Bước convert xảy ra **lúc build, không phải lúc chạy**:

- Thứ thực sự chạy trên server/browser **là file JS** — type đã bị xóa sạch, output gần như giống hệt JS viết tay. Runtime performance **bằng nhau**.
- Cái chậm hơn là **quy trình dev/build**: mỗi lần sửa code phải transpile (và type-check). Đây là chi phí trả một lần lúc build, đổi lấy việc bắt lỗi sớm.

## 2. So sánh `type` và `interface`

Với việc mô tả shape của object, hai cách gần như tương đương:

```typescript
interface User { id: number; name: string; }
type User = { id: number; name: string; };
```

Khác nhau ở phạm vi và hành vi:

| | `interface` | `type` |
|---|---|---|
| Mô tả được gì | Chỉ shape của object / function / class | **Mọi thứ**: object, union, primitive, tuple, mapped type... |
| Union | ❌ | ✅ `type Status = 'active' \| 'inactive'` |
| Alias primitive/tuple | ❌ | ✅ `type Id = number`, `type Point = [number, number]` |
| Kế thừa | `extends` (nhiều interface) | Giao kiểu bằng `&` (intersection) |
| Khai báo lại cùng tên | ✅ **Declaration merging** — các khai báo trùng tên tự gộp thành một | ❌ Trùng tên là lỗi |
| Class `implements` | ✅ | ✅ (trừ union type) |

Hai điểm khác biệt thực sự quan trọng:

**1. `type` làm được những thứ `interface` không làm được** — union là thứ dùng nhiều nhất:

```typescript
type BookingStatus = 'pending' | 'confirmed' | 'cancelled';
type Result = SuccessResponse | ErrorResponse;
```

**2. `interface` mở rộng được sau khi khai báo (declaration merging)** — chủ yếu dùng để vá kiểu của thư viện ngoài:

```typescript
// Thêm field vào Request của Express mà không sửa code thư viện
declare module 'express' {
  interface Request { venueId: number; }
}
```

Với `type` thì không thể: định nghĩa xong là đóng.

Các khác biệt tinh vi hơn (theo handbook chính thức và wiki Performance của TypeScript):

**3. Xử lý xung đột property:** `extends` phát hiện xung đột và **báo lỗi ngay**; intersection `&` chỉ merge đệ quy, property cùng tên khác kiểu **âm thầm** thành `never` — chạy đến chỗ dùng mới lộ:

```typescript
interface A { id: number; }
interface B extends A { id: string; }   // ❌ lỗi ngay tại đây

type C = { id: number } & { id: string }; // ✅ compile qua, nhưng id: never
```

**4. Hiệu năng compiler:** interface tạo ra một object type phẳng và quan hệ giữa các interface được **cache**; intersection phải merge đệ quy và tính lại mỗi lần kiểm tra. Codebase lớn với nhiều tầng kế thừa thì `extends` compile nhanh hơn `&` rõ rệt.

**5. Thông báo lỗi:** tên interface **luôn** hiện nguyên dạng trong error message; tên type alias chỉ **có thể** hiện (nhiều khi bị khai triển thành shape đầy đủ, khó đọc).

**Quy ước chọn** — handbook khuyến nghị: *"use `interface` until you need to use features from `type`"*:

- **Shape của object, contract cho class → `interface`:**

```typescript
// Shape dữ liệu đi qua các tầng
interface BookingPayload {
  venueId: number;
  partySize: number;
  startTime: Date;
}

// Contract cho class — mọi POS provider phải implement đủ
interface PosService {
  syncCustomers(venueId: number): Promise<void>;
  pushBooking(payload: BookingPayload): Promise<void>;
}

class WixPosService implements PosService { /* ... */ }
```

- **Union, alias primitive, tuple, kiểu dẫn xuất → `type`** (bắt buộc, interface không làm được):

```typescript
// Union — tập giá trị hữu hạn
type BookingStatus = 'pending' | 'confirmed' | 'cancelled';

// Union của nhiều shape
type PosEvent = BookingCreatedEvent | BookingCancelledEvent;

// Alias primitive / tuple
type VenueId = number;
type TimeRange = [Date, Date];

// Kiểu dẫn xuất từ kiểu có sẵn
type BookingUpdate = Partial<BookingPayload>;
type StatusLabels = Record<BookingStatus, string>;
```

- Quan trọng nhất là **thống nhất trong codebase** — trộn lẫn hai cách cho cùng một mục đích mới là vấn đề.

Nguồn: [Handbook — Everyday Types](https://www.typescriptlang.org/docs/handbook/2/everyday-types.html#differences-between-type-aliases-and-interfaces), [TypeScript wiki — Performance](https://github.com/microsoft/TypeScript/wiki/Performance#preferring-interfaces-over-intersections).

## 3. Tuple

Tuple là **mảng có độ dài cố định, mỗi vị trí có kiểu riêng** — khác array thường là mọi phần tử cùng một kiểu:

```typescript
type Ids = number[];             // array: bao nhiêu phần tử cũng được, đều là number
type TimeRange = [Date, Date];   // tuple: đúng 2 phần tử, cả hai là Date
type Entry = [string, number];   // tuple: vị trí 0 là string, vị trí 1 là number
```

Compiler kiểm tra cả **độ dài** lẫn **kiểu theo từng vị trí**:

```typescript
const range: TimeRange = [start, end];        // ✅
const bad1: TimeRange = [start];              // ❌ thiếu phần tử
const bad2: Entry = [1, 'a'];                 // ❌ sai kiểu theo vị trí
range[2];                                     // ❌ index ngoài độ dài tuple
```

Gặp nhiều nhất ở **giá trị trả về nhiều thành phần** (destructuring):

```typescript
function useState<T>(initial: T): [T, (v: T) => void] { /* ... */ }
const [count, setCount] = useState(0);   // count: number, setCount: (v: number) => void

// Object.entries trả về mảng tuple [string, T]
for (const [key, value] of Object.entries(config)) { /* ... */ }
```

Tuple còn hỗ trợ phần tử optional và rest: `[number, number?]`, `[string, ...number[]]`.

Lưu ý: tuple chỉ tồn tại ở tầng type — runtime nó vẫn là array JS bình thường, không có gì ngăn `push` thêm phần tử lúc chạy.
