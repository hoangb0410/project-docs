# Stripe — Lý thuyết cốt lõi và tra cứu SDK

Tài liệu lý thuyết Stripe cho backend, áp dụng được cho mọi project dùng Stripe làm cổng thanh toán. Trình bày theo thứ tự: mô hình → nền tảng API → SDK và các method chính → các object thanh toán → webhook → pattern thiết kế.

---

## 1. Stripe là gì — mô hình trong một câu

Stripe là **hệ thống xử lý thanh toán truy cập qua REST API**, trong đó mọi thứ — khách hàng, ý định thanh toán, thẻ, gói thuê bao, hóa đơn, refund — là **object có id riêng**; ứng dụng điều khiển tiền bằng cách tạo/đọc/cập nhật các object đó.

Hệ quả quan trọng nhất: **nguồn sự thật về tiền nằm ở Stripe, không nằm ở database của ta.** DB chỉ lưu id (`pi_...`, `cus_...`) và bản chiếu trạng thái; sự thật cuối cùng đến từ API retrieve hoặc webhook.

Ba đặc tính quyết định mọi cách dùng:

1. **Thanh toán là quá trình bất đồng bộ** — tạo intent → khách xác thực (3D Secure, redirect) → kết quả báo về qua webhook. Không bao giờ coi "gọi API xong" là "đã có tiền".
2. **Idempotency key** trên mọi request ghi — gửi lại cùng key thì Stripe trả kết quả cũ, không tạo giao dịch mới. Đây là công cụ chống double-charge khi retry.
3. **Tiền là số nguyên ở đơn vị nhỏ nhất** — `amount: 5000, currency: 'usd'` = 50.00 USD (cent). Không có số thực. Một số currency không có đơn vị lẻ (JPY, VND) thì amount = đúng mệnh giá.

---

## 2. Nền tảng API

### Id có tiền tố — đọc id biết ngay kiểu object

| Prefix  | Object           | Prefix             | Object          |
| ------- | ---------------- | ------------------ | --------------- |
| `cus_`  | Customer         | `sub_`             | Subscription    |
| `pi_`   | PaymentIntent    | `prod_` / `price_` | Product / Price |
| `seti_` | SetupIntent      | `in_`              | Invoice         |
| `pm_`   | PaymentMethod    | `acct_`            | Connect Account |
| `cs_`   | Checkout Session | `evt_`             | Event (webhook) |
| `ch_`   | Charge           | `re_`              | Refund          |

### Khóa và môi trường

| Khóa            | Prefix                  | Dùng ở                                                                   |
| --------------- | ----------------------- | ------------------------------------------------------------------------ |
| Secret key      | `sk_live_` / `sk_test_` | Server — toàn quyền, không bao giờ lộ ra client                          |
| Publishable key | `pk_live_` / `pk_test_` | Client — dựng UI thẻ (Elements); không cần nếu dùng Checkout hosted page |
| Webhook secret  | `whsec_`                | Verify chữ ký webhook — **mỗi endpoint một secret riêng**                |

Test mode và live mode là **hai thế giới dữ liệu tách biệt** — object bên test không tồn tại bên live.

### Các cơ chế chung cho mọi object

- **`metadata`** — map string→string (tối đa 50 key) ta tự do ghi lên hầu hết object. Dùng để nối ngược từ Stripe về domain (`{ orderId, userId }`). Nguyên tắc: chứa **reference id**, không chứa dữ liệu nghiệp vụ.
- **Idempotency key** — key phải **ổn định theo đơn vị nghiệp vụ** (`order-{id}-charge-{attempt}`), không phải random theo request. Stripe nhớ key trong 24h.
- **API version** — Stripe version hóa theo ngày (`2026-xx-xx`); SDK ghim version tại thời điểm phát hành SDK. **Nâng SDK = đổi API version ngầm** → nên pin `apiVersion` tường minh khi khởi tạo client.
- **`expand`** — API mặc định trả object liên quan dưới dạng id; `expand: ['customer', 'payment_intent.charges']` yêu cầu trả nguyên object lồng trong một lời gọi, đỡ N+1 request.
- **Pagination** — mọi API list dùng cursor: `limit` (mặc định 10, max 100) + `starting_after`; response có `has_more`.

---

## 3. SDK (stripe-node) — khởi tạo và quy ước gọi

### Khởi tạo

```typescript
import Stripe from "stripe";

const stripe = new Stripe(process.env.STRIPE_SECRET_KEY, {
  apiVersion: "2026-01-28", // pin tường minh — tránh đổi ngầm khi nâng SDK
  maxNetworkRetries: 2, // SDK tự retry lỗi mạng, an toàn vì có idempotency key tự sinh
  timeout: 20000, // ms
});
```

### Quy ước gọi

Mọi method theo dạng `stripe.<resource>.<action>(params, options?)`:

```typescript
const pi = await stripe.paymentIntents.create(
  { amount: 5000, currency: "usd", metadata: { orderId: "123" } }, // params — body của request
  { idempotencyKey: "order-123-charge-1", stripeAccount: "acct_x" }, // options — ngữ cảnh request
);
```

`options` (RequestOptions) quan trọng:

| Option           | Ý nghĩa                                                                 |
| ---------------- | ----------------------------------------------------------------------- |
| `idempotencyKey` | Chống tạo trùng khi retry (mục 2)                                       |
| `stripeAccount`  | Thực thi lời gọi trong ngữ cảnh một connected account (Connect — mục 6) |
| `apiVersion`     | Override version cho riêng lời gọi này                                  |

### Duyệt list và xử lý lỗi

```typescript
// Auto-pagination — SDK tự lật trang
for await (const customer of stripe.customers.list({ limit: 100 })) { ... }

// Lỗi có phân loại
try { ... } catch (err) {
  if (err instanceof Stripe.errors.StripeCardError) { }        // thẻ bị từ chối — err.code, err.decline_code
  if (err instanceof Stripe.errors.StripeInvalidRequestError) { } // params sai — bug của ta
  if (err instanceof Stripe.errors.StripeRateLimitError) { }   // quá tải — backoff rồi retry
}
```

---

## 4. Các object thanh toán và method chính

### 4.1. Customer — hồ sơ người trả tiền

Neo thẻ đã lưu và subscription. Không có Customer thì mỗi lần thanh toán phải nhập thẻ từ đầu.

| Method                                                        | Tham số chính                                                                                      |
| ------------------------------------------------------------- | -------------------------------------------------------------------------------------------------- |
| `customers.create(params)`                                    | `email`, `name`, `phone`, `metadata`, `payment_method` + `invoice_settings.default_payment_method` |
| `customers.retrieve(id)` / `.update(id, params)` / `.del(id)` | —                                                                                                  |
| `customers.list(params)`                                      | `email` (tìm theo email), `limit`, `starting_after`                                                |

### 4.2. PaymentIntent — một lần thu tiền, dạng máy trạng thái

Object trung tâm của mọi lần thu tiền. Vòng đời:

```
requires_payment_method → requires_confirmation → requires_action → processing → succeeded
                                                                              ↘ canceled
```

Backend tạo intent rồi trả **`client_secret`** cho client hoàn tất với Stripe — amount đã bị khóa từ server, client không bao giờ quyết định số tiền. Kết quả cuối đến qua webhook, không qua response của lời gọi tạo.

| Method                               | Tham số chính                                                                                                                                                                                                                                                                                                                                 |
| ------------------------------------ | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `paymentIntents.create(params)`      | `amount`_, `currency`_, `customer`, `payment_method`, `confirm` (tạo + xác nhận luôn), `off_session` (charge thẻ đã lưu, không có mặt khách), `capture_method: 'automatic' \| 'manual'` (manual = authorize giữ tiền, capture sau), `setup_future_usage: 'off_session'` (thu tiền + lưu thẻ luôn), `metadata`, `description`, `receipt_email` |
| `paymentIntents.retrieve(id)`        | `expand`                                                                                                                                                                                                                                                                                                                                      |
| `paymentIntents.confirm(id, params)` | `payment_method`                                                                                                                                                                                                                                                                                                                              |
| `paymentIntents.capture(id, params)` | `amount_to_capture` — chốt tiền của intent `capture_method: manual` (mặc định giữ được 7 ngày)                                                                                                                                                                                                                                                |
| `paymentIntents.cancel(id)`          | nhả tiền đang authorize                                                                                                                                                                                                                                                                                                                       |

Hai mô hình "giữ chân khách" cần phân biệt:

- **Authorization** (`capture_method: 'manual'`) — **giữ tiền thật** trên thẻ, capture trong ~7 ngày hoặc mất.
- **Card on file** (SetupIntent, mục 4.3) — **chỉ lưu thẻ**, không giữ đồng nào; charge sau bằng `off_session`.

Charge off-session thẻ đã lưu:

```typescript
await stripe.paymentIntents.create(
  {
    amount,
    currency,
    customer: "cus_x",
    payment_method: "pm_x",
    off_session: true,
    confirm: true,
  },
  { idempotencyKey },
);
// có thể ném lỗi authentication_required — 3DS không làm được khi vắng khách
```

### 4.3. SetupIntent — lưu thẻ, chưa thu tiền

Cùng họ PaymentIntent nhưng kết quả là một **PaymentMethod gắn vào Customer** để charge sau.

| Method                                               | Tham số chính                                                          |
| ---------------------------------------------------- | ---------------------------------------------------------------------- |
| `setupIntents.create(params)`                        | `customer`, `payment_method_types`, `usage: 'off_session'`, `metadata` |
| `setupIntents.retrieve(id)` / `.confirm(id, params)` | —                                                                      |

### 4.4. PaymentMethod — cái thẻ

| Method                                                    | Tham số chính                |
| --------------------------------------------------------- | ---------------------------- |
| `paymentMethods.attach(id, { customer })` / `.detach(id)` | gắn / gỡ thẻ khỏi Customer   |
| `paymentMethods.list(params)`                             | `customer`\*, `type: 'card'` |

Dữ liệu thẻ thật (số thẻ, CVC) **không bao giờ chạm server của ta** — client trao đổi trực tiếp với Stripe (Elements/Checkout), server chỉ thấy id `pm_` và 4 số cuối.

### 4.5. Refund

| Method                                               | Tham số chính                                                                                                                                    |
| ---------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------ |
| `refunds.create(params)`                             | `payment_intent` hoặc `charge`, `amount` (bỏ trống = hoàn toàn bộ), `reason: 'requested_by_customer' \| 'duplicate' \| 'fraudulent'`, `metadata` |
| `refunds.retrieve(id)` / `.list({ payment_intent })` | —                                                                                                                                                |

Refund là thao tác ghi rủi ro retry cao nhất (hay chạy trong nhánh lỗi/rollback) → **luôn kèm idempotencyKey**.

### 4.6. Ngưỡng tối thiểu

Stripe từ chối charge dưới ~0.50 USD tương đương mỗi currency (GBP 30p, EUR 50c...). Thiết kế phải trả lời "tổng dưới ngưỡng thì sao" — thường là bỏ qua thanh toán thay vì báo lỗi.

---

## 5. Checkout Session — trang thanh toán do Stripe host

Thay vì tự dựng form thẻ (đụng PCI compliance), backend tạo một Session mô tả "thu gì, bao nhiêu, xong về đâu" và nhận URL; khách được redirect sang trang Stripe, xong Stripe redirect về `success_url` / `cancel_url`.

| Method                                | Tham số chính                                |
| ------------------------------------- | -------------------------------------------- |
| `checkout.sessions.create(params)`    | xem bảng dưới                                |
| `checkout.sessions.retrieve(id)`      | `expand: ['payment_intent', 'setup_intent']` |
| `checkout.sessions.expire(id)`        | đóng session đang mở                         |
| `checkout.sessions.listLineItems(id)` | đọc lại các dòng hàng                        |

Tham số `create` quan trọng:

| Tham số                                             | Ý nghĩa                                                                                                                                                |
| --------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `mode`\*                                            | `'payment'` (thu một lần → PaymentIntent) / `'setup'` (lưu thẻ → SetupIntent) / `'subscription'` (→ Subscription)                                      |
| `line_items`                                        | mảng `{ quantity, price }` hoặc `{ quantity, price_data: { currency, unit_amount, product_data: { name } } }` — giá ad-hoc không cần tạo Product trước |
| `success_url` / `cancel_url`                        | nơi redirect về; `success_url` có thể chứa `{CHECKOUT_SESSION_ID}`                                                                                     |
| `customer` / `customer_email` / `customer_creation` | gắn Customer sẵn có / điền sẵn email / `'always'` để Stripe tự tạo Customer                                                                            |
| `payment_intent_data`                               | truyền xuống PaymentIntent con: `metadata`, `setup_future_usage: 'off_session'` (vừa thu vừa lưu thẻ)                                                  |
| `setup_intent_data`                                 | truyền `metadata` xuống SetupIntent con (mode setup)                                                                                                   |
| `expires_at`                                        | mặc định 24h, tối thiểu 30 phút                                                                                                                        |
| `metadata`                                          | metadata của chính session                                                                                                                             |

Ví dụ — thu tiền một lần (đơn hàng có 2 dòng, vừa thu vừa lưu thẻ):

```typescript
const session = await stripe.checkout.sessions.create(
  {
    mode: "payment",
    line_items: [
      {
        quantity: 1,
        price_data: {
          currency: "usd",
          unit_amount: 5000,
          product_data: { name: "Tiền cọc" },
        },
      },
      {
        quantity: 2,
        price_data: {
          currency: "usd",
          unit_amount: 1500,
          product_data: { name: "Phụ thu" },
        },
      },
    ],
    customer_creation: "always", // Stripe tự tạo Customer
    payment_intent_data: {
      metadata: { orderId: "123" }, // để webhook nối ngược về domain
      setup_future_usage: "off_session", // lưu thẻ để charge sau
    },
    success_url:
      "https://app.example.com/orders/123/success?session_id={CHECKOUT_SESSION_ID}",
    cancel_url: "https://app.example.com/orders/123/cancel",
  },
  { idempotencyKey: "order-123-checkout-1" },
);

// redirect khách sang session.url — phần còn lại Stripe lo
// kết quả cuối cùng nhận qua webhook checkout.session.completed / payment_intent.succeeded
```

Ví dụ — chỉ lưu thẻ, không thu tiền (`mode: 'setup'`):

```typescript
const session = await stripe.checkout.sessions.create({
  mode: "setup",
  customer: "cus_x",
  setup_intent_data: { metadata: { orderId: "123" } },
  success_url: "https://app.example.com/orders/123/card-saved",
  cancel_url: "https://app.example.com/orders/123/cancel",
});
// webhook setup_intent.succeeded trả về payment_method đã gắn vào Customer
```

Hai đặc tính vòng đời phải thiết kế trước: session **hết hạn** và khách **có thể bỏ ngang** — hệ thống phải định nghĩa trạng thái đơn hàng khi session mồ côi. Pattern hay dùng: **reuse-or-create** — còn session mở với tổng tiền không đổi thì trả lại URL cũ thay vì tạo mới.

Họ hàng cùng triết lý "Stripe host UI":

- `billingPortal.sessions.create({ customer, return_url, flow_data })` — cổng tự phục vụ: đổi thẻ, xem hóa đơn, hủy gói.
- Payment Links — link thanh toán tĩnh tạo sẵn, không cần code.

---

## 6. Stripe Connect — thu tiền hộ người khác

Bài toán platform/marketplace: khách trả tiền cho **merchant** (nhà hàng, người bán), không phải cho platform. Connect giải bằng **connected account** — mỗi merchant một tài khoản Stripe con.

### Ba loại account

| Loại        | Ai chịu onboarding/compliance/dashboard                     | Khi nào chọn                        |
| ----------- | ----------------------------------------------------------- | ----------------------------------- |
| Standard    | Merchant tự quản, có dashboard Stripe đầy đủ                | Merchant rành Stripe                |
| **Express** | Stripe host onboarding + payout, platform quản phần còn lại | Cân bằng — lựa chọn phổ biến nhất   |
| Custom      | Platform tự làm toàn bộ UI + gánh compliance                | Cần kiểm soát tuyệt đối trải nghiệm |

### Onboarding — Account Links

```typescript
const account = await stripe.accounts.create({
  type: "express",
  country: "AU",
  capabilities: {
    card_payments: { requested: true },
    transfers: { requested: true },
  },
});
const link = await stripe.accountLinks.create({
  account: account.id,
  type: "account_onboarding",
  refresh_url,
  return_url, // link dùng một lần, hết hạn nhanh
});
// redirect merchant sang link.url
```

Không có callback "đã xong" đáng tin — trạng thái thật đọc bằng `accounts.retrieve(id)` và nhìn **`charges_enabled`**, **`payouts_enabled`**, **`requirements.currently_due`**. Mọi tính năng thu tiền phải gate trên trạng thái này.

### Ba kiểu charge — quyết định ai chịu phí, refund, dispute

| Kiểu                         | Cách gọi                                                             | Tiền/charge thuộc về                                          |
| ---------------------------- | -------------------------------------------------------------------- | ------------------------------------------------------------- |
| **Direct**                   | option `{ stripeAccount: 'acct_x' }` trên lời gọi                    | Connected account — merchant chịu phí Stripe, refund, dispute |
| Destination                  | params `transfer_data: { destination }` (+ `application_fee_amount`) | Platform nhận rồi chuyển — platform chịu phí/dispute          |
| Separate charges & transfers | thu trên platform, tự `transfers.create` chia sau                    | Linh hoạt nhất, phức tạp nhất                                 |

Với direct charge, **mọi** lời gọi thuộc luồng đó (create session, retrieve, refund, webhook construct) đều phải kèm đúng `stripeAccount` — object nằm trên account con, không thấy được từ dashboard platform. Platform muốn thu phí thì thêm `application_fee_amount` trên intent/session.

---

## 7. Billing — Subscription và họ hàng

Mô hình xếp tầng: **Product** (bán gì) → **Price** (giá nào, chu kỳ nào — một product nhiều price) → **Subscription** (customer nào mua price nào) → **Invoice** (tự sinh mỗi chu kỳ) → thanh toán.

| Method                                          | Tham số chính                                                                                                                      |
| ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `products.create/retrieve/update/list`          | `name`, `active`, `metadata`                                                                                                       |
| `prices.create/retrieve/list`                   | `product`, `unit_amount`, `currency`, `recurring: { interval: 'month' \| 'year' }`, `lookup_key`                                   |
| `subscriptions.create(params)`                  | `customer`\*, `items: [{ price, quantity }]`, `trial_period_days`, `payment_behavior: 'default_incomplete'`                        |
| `subscriptions.update(id, params)`              | `items` (đổi gói), `proration_behavior: 'always_invoice' \| 'create_prorations' \| 'none'`, `cancel_at_period_end: true` (hủy mềm) |
| `subscriptions.cancel(id)`                      | hủy ngay                                                                                                                           |
| `invoices.list({ customer })` / `.retrieve(id)` | —                                                                                                                                  |
| `subscriptionItems.create/del`                  | thêm/bớt item trong gói đang chạy                                                                                                  |

Usage-based billing: khai **Billing Meter**, ứng dụng bắn `billing.meterEvents.create({ event_name, payload: { stripe_customer_id, value } })`, Stripe tự tính tiền cuối kỳ theo meter price.

Trạng thái subscription cần xử lý: `trialing`, `active`, `past_due` (thu tiền fail, đang retry), `canceled`, `incomplete`. Nguồn cập nhật duy nhất đáng tin: webhook (`customer.subscription.updated/deleted`, `invoice.payment_succeeded/failed`). Lưu ý lọc `invoice.billing_reason === 'subscription_cycle'` nếu chỉ muốn bắt sự kiện gia hạn.

---

## 8. Webhook — kênh sự thật từ Stripe về

Mọi biến động sinh một **Event** `{ id: 'evt_...', type: 'payment_intent.succeeded', data: { object: {...} } }`. Stripe POST về endpoint đã đăng ký; không nhận 2xx thì **retry với backoff trong nhiều ngày**.

### Ba luật bắt buộc

1. **Verify chữ ký trên raw body.**

```typescript
// Express/NestJS: raw body parser CHỈ cho path webhook, đặt trước JSON parser
app.use("/webhook/stripe", express.raw({ type: "application/json" }));

const event = stripe.webhooks.constructEvent(
  req.body,
  req.headers["stripe-signature"],
  whsec,
);
// sai chữ ký → ném lỗi → trả 400
```

Bẫy kinh điển: JSON parser toàn cục parse mất raw body → chữ ký không bao giờ khớp.

2. **Idempotent.** Cùng event có thể đến nhiều lần (retry, mạng trùng). Dedupe theo `event.id` — chuẩn nhất là insert vào bảng có UNIQUE trên `event_id`, đụng unique constraint thì trả "already processed" (check-then-act atomic ở tầng DB). Thêm chốt chặn thứ hai theo trạng thái nghiệp vụ: đơn đã ở trạng thái terminal thì bỏ qua.

3. **Không giả định thứ tự.** Event có thể đến lộn xộn — handler kiểm trạng thái hiện tại trước khi chuyển, không suy diễn từ thứ tự nhận.

### Quy tắc xử lý

- **Trả 2xx nhanh** — xử lý nặng đẩy ra queue/sau response; handler chậm quá timeout bị Stripe tính là fail và retry.
- Chuyển trạng thái nghiệp vụ trong transaction; side effect (gửi mail, gọi hệ thống khác) chạy **sau commit** — sự kiện tiền không được rollback theo lỗi của side effect.
- **Đăng ký event type trên dashboard** cho endpoint — code có handler mà dashboard không gửi thì không bao giờ chạy.
- Với Connect: event của connected account về endpoint riêng ("listen to connected accounts"), event mang thêm trường `account`.

### Các event type hay dùng

| Event                                                        | Nghĩa                                                           |
| ------------------------------------------------------------ | --------------------------------------------------------------- |
| `payment_intent.succeeded` / `payment_intent.payment_failed` | Thu tiền xong / thất bại                                        |
| `setup_intent.succeeded` / `setup_intent.setup_failed`       | Lưu thẻ xong / thất bại                                         |
| `checkout.session.completed`                                 | Khách hoàn tất trang Checkout (rẽ nhánh theo `mode`)            |
| `checkout.session.expired`                                   | Session mồ côi hết hạn — chỗ dọn đơn treo                       |
| `charge.refunded`                                            | Có refund (đọc `amount_refunded` để phân biệt một phần/toàn bộ) |
| `charge.dispute.created`                                     | Khách kiện chargeback                                           |
| `invoice.payment_succeeded` / `invoice.payment_failed`       | Chu kỳ subscription thu được / không thu được                   |
| `customer.subscription.updated` / `.deleted`                 | Gói thay đổi / hủy                                              |
| `account.updated`                                            | Trạng thái connected account đổi (charges_enabled...)           |

### Đừng tin success_url

Khách quay về `success_url` **không chứng minh** đã thanh toán — URL đoán được, khách có thể đóng tab trước redirect, redirect có thể đến trước hoặc sau webhook. Quy tắc: **redirect chỉ để hiển thị UI; chuyển trạng thái nghiệp vụ chỉ theo webhook.**

---

## 9. Các pattern thiết kế xuyên suốt

1. **DB lưu id + trạng thái chiếu, không lưu dữ liệu thẻ.** Cột điển hình: `payment_intent_id`, `customer_id`, `refund_id`, `idempotency_key`, `payment_status`, kèm cột JSON cho response phụ. Đặt tên cột trung lập (`provider_*`) nếu tương lai có thể thêm cổng thanh toán khác.
2. **Amount luôn tính server-side.** Client gửi "mua gì", server tính "bao nhiêu tiền". Nhận amount từ client = lỗ hổng sửa giá.
3. **Metadata trên mọi object tạo ra** — gắn reference id domain (`orderId`) ngay lúc tạo để webhook handler nối ngược được.
4. **Idempotency key theo đơn vị nghiệp vụ** cho mọi thao tác ghi, đặc biệt là thao tác chạy trong nhánh lỗi/retry (refund tự động).
5. **Máy trạng thái phía ta chỉ tiến, không lùi** — `pending → paid → refunded`; webhook đến muộn/trùng không được kéo trạng thái ngược lại.
6. **Degrade có chủ đích** khi Stripe không khả dụng hoặc account chưa sẵn sàng — quyết định trước là chặn nghiệp vụ hay cho đi tiếp không thanh toán, đừng để ngẫu nhiên.
7. **Test bằng Stripe CLI**: `stripe listen --forward-to localhost:3000/webhook` để nhận webhook local, `stripe trigger payment_intent.succeeded` để bắn event giả, bộ thẻ test `4242 4242 4242 4242` (thành công) / `4000 0000 0000 9995` (từ chối) / `4000 0025 0000 3155` (đòi 3DS).

---

## 10. Checklist khi thêm một thao tác Stripe mới

1. Thao tác ghi? → idempotency key ổn định theo đơn vị nghiệp vụ đã có chưa?
2. Amount lấy từ đâu? → server tính, đơn vị nhỏ nhất, đã so ngưỡng tối thiểu theo currency chưa?
3. Có bước bất đồng bộ (khách xác thực, invoice chạy)? → trạng thái nghiệp vụ chuyển ở **webhook**, không phải ở response API hay redirect.
4. Webhook handler mới: verify chữ ký raw body? dedupe theo `event.id`? chốt theo trạng thái hiện tại? đã đăng ký event type trên dashboard?
5. Object mới cần truy ngược về domain? → gắn `metadata` reference id lúc tạo, lưu id Stripe vào DB.
6. Dùng Connect? → lời gọi đã kèm đúng `stripeAccount` chưa? account đã `charges_enabled` chưa?
7. Nếu Stripe sập hoặc thanh toán treo, nghiệp vụ degrade thế nào?

Câu số 3 là quan trọng nhất: mọi bug thanh toán kinh điển — double charge, trạng thái ma, tiền về mà đơn không mở — đều quy về việc coi một quá trình bất đồng bộ như thể nó đồng bộ.
