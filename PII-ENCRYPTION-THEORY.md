# Mã hóa dữ liệu PII & Search trên dữ liệu mã hóa

Tài liệu lý thuyết về bài toán bảo vệ dữ liệu cá nhân (PII) trong ứng dụng: tại sao phải mã hóa, có những lựa chọn thuật toán nào, AES-256-CBC hoạt động ra sao, và làm thế nào để search được trên dữ liệu đã mã hóa (blind index).

Các khối **📌 Ví dụ thực tế** minh họa bằng codebase nollie-api — một hệ thống CRM/booking cho nhà hàng có khách hàng ở UK — nhưng lý thuyết áp dụng cho mọi hệ thống lưu PII.

---

## 1. Vấn đề

### 1.1. PII và yêu cầu pháp lý

**PII (Personally Identifiable Information)** là dữ liệu định danh được một cá nhân: họ tên, email, số điện thoại, ngày sinh, địa chỉ... Hầu hết các khung pháp lý về bảo vệ dữ liệu — **GDPR** (EU), **UK GDPR + Data Protection Act 2018** (UK), CCPA (California)... — đều yêu cầu "biện pháp kỹ thuật phù hợp" để bảo vệ loại dữ liệu này, và **encryption được nêu đích danh** (GDPR Điều 32) là một biện pháp như vậy.

Động lực thực dụng nhất: nếu DB bị lộ (dump SQL, backup thất lạc, truy cập trái phép) mà PII đã được mã hóa với key nằm ngoài DB, thì dữ liệu lộ ra chỉ là ciphertext vô nghĩa — theo GDPR, sự cố có thể không còn bị xếp vào nhóm breach nghiêm trọng nhất phải thông báo tới từng người dùng.

### 1.2. Mã hóa ở tầng nào?

Có 3 tầng có thể đặt mã hóa, bảo vệ được các mối đe dọa khác nhau:

| Tầng               | Ví dụ                                           | Chống được                                                | KHÔNG chống được                        |
| ------------------ | ----------------------------------------------- | --------------------------------------------------------- | --------------------------------------- |
| **Disk (at rest)** | RDS encryption, LUKS                            | Trộm ổ cứng vật lý, snapshot bị lộ                        | Bất kỳ ai chạy được SQL                 |
| **DB engine**      | pgcrypto, TDE, SQL Server Always Encrypted      | Một phần truy cập SQL (tùy cấu hình)                      | Key thường vẫn nằm trong tầm với của DB |
| **Application**    | App tự encrypt trước INSERT, decrypt sau SELECT | SQL injection, developer/BI tool đọc thẳng DB, dump logic | Bug ở chính app, lộ key ở app server    |

Điểm mấu chốt của **application-level encryption**: trong DB chỉ tồn tại ciphertext, key nằm ở app server (env var / secret manager), **tách hoàn toàn khỏi DB**. Kẻ có toàn quyền SQL vẫn không đọc được gì. Đây là tầng duy nhất thỏa yêu cầu "DB dump không lộ PII", và là chủ đề của tài liệu này. (Thực tế nên dùng chồng lớp: at-rest luôn bật, application-level thêm cho các field nhạy cảm.)

### 1.3. Mâu thuẫn cốt lõi: mã hóa vs search

Đây là bài toán khó nhất của mã hóa field-level:

- Mã hóa **tốt** nghĩa là ciphertext trông như nhiễu ngẫu nhiên — không lộ bất kỳ thông tin gì về plaintext, kể cả việc hai giá trị có bằng nhau hay không.
- Nhưng nghiệp vụ luôn cần `WHERE email LIKE 'john%'` — người dùng gõ vài ký tự để tìm một bản ghi.
- Hai yêu cầu này **mâu thuẫn trực tiếp về mặt toán học**: nếu ciphertext không mang thông tin gì về plaintext thì không tồn tại phép so khớp nào giữa nó và từ khóa search.

Hệ quả: mọi giải pháp "searchable encryption" thực tế đều là **trade-off** — cố ý leak một lượng thông tin có kiểm soát (equality, prefix...) để đổi lấy khả năng search. Việc thiết kế nằm ở chỗ chọn leak gì và chấp nhận đến đâu. Phần 4 trình bày lời giải phổ biến nhất trong thực tế: **blind index**.

### 1.4. Các ràng buộc triển khai thường gặp

Ba ràng buộc xuất hiện ở hầu hết hệ thống thật, định hình thiết kế:

1. **Mã hóa có điều kiện** — chỉ một nhóm dữ liệu/region chịu quy định (ví dụ chỉ khách EU/UK). Giải pháp thường là feature flag theo deployment thay vì hardcode logic region.
2. **Dữ liệu cũ chưa mã hóa** — hệ thống đang chạy không thể mã hóa toàn bộ trong một đêm. Phía đọc phải "sống chung" được với cả plaintext lẫn ciphertext trong cùng một cột suốt giai đoạn migrate dần.
3. **Mã hóa DB nhưng log ra plaintext thì vô nghĩa** — chính sách "không log PII" phải đi kèm, nếu không log file trở thành bản sao không mã hóa của chính dữ liệu cần bảo vệ.

> **📌 Ví dụ thực tế — nollie-api:** hệ thống chạy 2 region AU/NZ và UK; chỉ UK chịu UK GDPR nên toàn bộ logic mã hóa gate bằng env `ENABLE_ENCRYPTION === 'true'` theo deployment ([src/configs/env-config.ts](src/configs/env-config.ts)). Dữ liệu cũ backfill dần qua queue, nên service đọc phải xử lý được cột lẫn lộn 2 loại giá trị. Quy tắc codebase: không bao giờ log email/phone của khách UK, chỉ log ID.

---

## 2. Có những lựa chọn thuật toán nào?

### 2.1. Phân loại tổng quan

Yêu cầu của bài toán PII: dữ liệu phải **đọc lại được** (hiển thị email lên UI), mã hóa/giải mã phải **nhanh** (mỗi query đụng hàng chục field), và lý tưởng là **search được** (giải quyết riêng ở phần 4).

| Hướng                      | Ví dụ                   | Đọc lại được?    | Tốc độ             | Phù hợp PII?                                                          |
| -------------------------- | ----------------------- | ---------------- | ------------------ | --------------------------------------------------------------------- |
| **Hashing (một chiều)**    | SHA-256, bcrypt, Argon2 | ❌               | nhanh              | ❌ — chỉ hợp cho password (không bao giờ cần đọc lại)                 |
| **Mã hóa đối xứng**        | AES, ChaCha20           | ✅ (cùng 1 key)  | rất nhanh          | ✅ — lựa chọn chuẩn                                                   |
| **Mã hóa bất đối xứng**    | RSA, ECC                | ✅ (private key) | chậm hơn 100–1000× | ❌ — thiết kế cho trao đổi key/chữ ký số, không phải mã hóa hàng loạt |
| **Homomorphic (FHE)**      | Microsoft SEAL          | ✅               | chậm hơn 10⁴–10⁶×  | ❌ — tính toán được trên ciphertext nhưng chưa thực tế cho CRUD       |
| **Order-preserving (OPE)** | CryptDB-style           | ✅               | nhanh              | ⚠️ — cho range/sort nhưng leak thứ tự, đã bị phá trong thực tế        |

→ Với PII, **mã hóa đối xứng là lựa chọn duy nhất hợp lý**. Câu hỏi còn lại là: thuật toán nào, key bao nhiêu bit, mode gì.

### 2.2. Trong họ đối xứng: tại sao AES?

- **AES (Advanced Encryption Standard)** là chuẩn NIST từ 2001, được cộng đồng mật mã học phân tích liên tục hơn 20 năm mà chưa có tấn công thực tế nào vào bản thân thuật toán. Nó là mặc định của cả ngành — "không ai bị đuổi việc vì chọn AES".
- CPU server hiện đại có **AES-NI** (tập lệnh phần cứng chuyên dụng) → chi phí mã hóa gần như không đáng kể so với I/O của một query.
- Mọi ngôn ngữ/runtime đều hỗ trợ native (Node.js: module `crypto` trên nền OpenSSL) — không thêm dependency.
- Đối thủ đáng kể duy nhất là **ChaCha20-Poly1305** — nhanh hơn trên thiết bị _không_ có AES-NI (mobile, IoT); trên server x86 thì AES thắng.

### 2.3. Key size: tại sao 256?

AES có 3 cỡ key: 128 / 192 / 256 bit. AES-128 đến nay vẫn chưa phá được, nhưng chuẩn thực hành nghiêng về 256 vì:

- **Security margin** lớn hơn (14 rounds so với 10).
- **Kháng lượng tử**: thuật toán Grover trên máy tính lượng tử giảm độ mạnh hiệu dụng của khóa đối xứng còn một nửa — 256 → 128 bit (vẫn an toàn), 128 → 64 bit (không còn an toàn).
- Chi phí thêm ~40% trên một nền "gần như miễn phí" → không đáng kể.
- Các hướng dẫn compliance (kể cả của ICO — cơ quan bảo vệ dữ liệu UK) thường lấy AES-256 làm chuẩn tham chiếu.

### 2.4. Mode of operation: ECB / CBC / CTR / GCM

AES tự thân chỉ mã hóa được đúng **1 block 16 bytes**. Dữ liệu dài hơn cần một **mode of operation** để xâu chuỗi các block — và chọn mode quan trọng không kém chọn thuật toán:

| Mode    | Cơ chế                                                         | Đánh giá                                                                                                                      |
| ------- | -------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------- |
| **ECB** | Mỗi block mã hóa độc lập                                       | ❌ Block plaintext giống nhau → ciphertext giống nhau, lộ cấu trúc dữ liệu (ảnh "ECB penguin" kinh điển). Không bao giờ dùng. |
| **CBC** | Block sau XOR với ciphertext block trước; block đầu XOR với IV | ✅ Che pattern. ⚠️ Không có kiểm tra toàn vẹn (ciphertext sửa được mà không bị phát hiện), cần padding                        |
| **CTR** | Biến block cipher thành stream cipher bằng bộ đếm              | ✅ Song song hóa, không cần padding. ⚠️ Cũng không toàn vẹn; tái sử dụng nonce là thảm họa                                    |
| **GCM** | CTR + tag xác thực (AEAD — Authenticated Encryption)           | ✅ Chuẩn hiện đại: vừa bảo mật vừa chống sửa đổi. **Mặc định cho hệ thống xây mới**                                           |

**Khi nào CBC vẫn là lựa chọn hợp lý?** Khi hệ thống cần **deterministic encryption** — cùng plaintext luôn cho cùng ciphertext (xem 3.4) — để so khớp equality hoặc dedupe trên cột mã hóa. GCM với nonce ngẫu nhiên _cố tình_ phá tính chất này (đó là ưu điểm bảo mật của nó). Ngoài ra, nếu threat model chỉ là _lộ dữ liệu khi DB bị dump_ chứ không phải _kẻ tấn công chủ động sửa ciphertext trong DB_, thì auth tag của GCM không mang lại giá trị tương xứng với chi phí lưu thêm tag + nonce cạnh mỗi giá trị.

> **📌 Ví dụ thực tế — nollie-api:** chọn AES-256-CBC với IV tĩnh, chấp nhận không có authenticated encryption, vì (1) threat model là DB dump, ai sửa được DB thì đã thắng ở nơi khác rồi; (2) cần deterministic để so khớp và idempotent khi sync lại từ POS. Đây là trade-off có chủ đích, không phải thiếu sót do không biết GCM.

---

## 3. Chi tiết AES-256-CBC

### 3.1. AES hoạt động thế nào (mức khái niệm)

AES là **block cipher**: nhận 1 block **128 bit (16 bytes)** plaintext + key → 16 bytes ciphertext. Với key 256-bit nó chạy **14 rounds**; mỗi round biến đổi một ma trận 4×4 bytes qua 4 phép:

1. **SubBytes** — thay từng byte qua bảng S-box (phi tuyến — nguồn "confusion")
2. **ShiftRows** — dịch vòng các hàng (khuếch tán theo hàng)
3. **MixColumns** — nhân ma trận trên trường GF(2⁸) (khuếch tán theo cột)
4. **AddRoundKey** — XOR với round key (sinh từ key gốc qua key schedule)

Hai tính chất đích của mọi cipher tốt, theo Shannon:

- **Confusion** — quan hệ giữa key và ciphertext phức tạp đến mức không phân tích thống kê được.
- **Diffusion** — đổi 1 bit input → xấp xỉ 50% bit output đổi theo (avalanche effect). Chính diffusion là lý do không thể prefix-search trên ciphertext (xem 4.1).

### 3.2. Tham số của AES-256-CBC

Một phép mã hóa AES-256-CBC là hàm 3 đầu vào → 1 đầu ra:

```
ciphertext = AES-256-CBC(key, iv, plaintext)
plaintext  = AES-256-CBC⁻¹(key, iv, ciphertext)    // giải mã cần đúng cả key lẫn IV
```

| Tham số   | Kích thước                                                | Bí mật?             | Vai trò                                                                                                                                                                            |
| --------- | --------------------------------------------------------- | ------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Key**   | **32 bytes (256 bit)** — con số "256" trong tên           | ✅ Tuyệt đối bí mật | Toàn bộ độ an toàn nằm ở đây. Lộ key = lộ tất cả dữ liệu. Sinh ngẫu nhiên (CSPRNG), lưu ở secret manager / env var, **không bao giờ** nằm trong DB hay source code                 |
| **IV**    | **16 bytes (128 bit)** — bằng block size, bất kể key size | ❌ Không cần bí mật | Giá trị "mồi" cho block đầu tiên của chuỗi CBC (xem 3.3). Quyết định cùng plaintext có ra cùng ciphertext hay không — random per-record (chuẩn) vs static (deterministic), xem 3.4 |
| Plaintext | bất kỳ                                                    | —                   | Được đệm PKCS#7 lên bội của 16 bytes trước khi mã hóa                                                                                                                              |

Hai điểm hay nhầm:

- **IV dài 16 bytes chứ không phải 32** — kích thước IV theo _block size_ của AES (luôn 128 bit), không theo key size. AES-128/192/256 đều dùng IV 16 bytes.
- **Key phải đúng chằn chặn 32 bytes** — Node.js `createCipheriv('aes-256-cbc', key, iv)` throw nếu key/IV sai độ dài, không tự pad. Nếu key nguồn là chuỗi tùy ý (passphrase), chuẩn mực là dẫn xuất qua KDF (PBKDF2, scrypt, HKDF) thay vì pad thô.

> **📌 Ví dụ thực tế — nollie-api:** hai tham số này map thẳng vào 2 env var — `ENCRYPTION_KEY` (pad/cắt về 32 bytes bằng `padEnd(32, '0').substring(0, 32)` — cách làm thô, đủ dùng khi env đã là chuỗi ngẫu nhiên đủ dài, nhưng kém chuẩn hơn KDF) và `ENCRYPTION_IV` (bắt buộc đúng 32 hex chars = 16 bytes, validate cứng lúc boot, sai là throw). Đây chính là 2 biến trong file `.env`.

### 3.3. CBC — Cipher Block Chaining

Plaintext cắt thành các block 16 bytes, xâu chuỗi:

```
C₁ = AES(K, P₁ ⊕ IV)
C₂ = AES(K, P₂ ⊕ C₁)
C₃ = AES(K, P₃ ⊕ C₂)
...
```

- Mỗi block plaintext bị XOR với ciphertext của block trước → hai block giống nhau cho ra ciphertext khác nhau (chính là điểm ECB thiếu).
- Block đầu không có "ciphertext trước" nên cần **IV (Initialization Vector)** — 16 bytes — đóng vai trò đó. IV không cần bí mật, nhưng cách chọn IV quyết định tính chất bảo mật (3.4).
- **Padding PKCS#7**: plaintext hiếm khi chia hết cho 16; block cuối được đệm N bytes, mỗi byte mang giá trị N. Hệ quả: ciphertext luôn là bội của 16 bytes → nếu encode hex, luôn là bội của 32 ký tự. (Tính chất này hữu ích để _nhận diện_ một giá trị có phải ciphertext không — xem ví dụ 3.5.)

### 3.4. IV: random (chuẩn) vs static (deterministic)

**Chuẩn lý thuyết**: IV phải **ngẫu nhiên, mới cho mỗi lần mã hóa**, lưu kèm ciphertext. Khi đó cùng một plaintext mã hóa 2 lần ra 2 ciphertext khác nhau — đạt _semantic security_, không leak gì kể cả việc 2 bản ghi có giá trị bằng nhau.

**Biến thể deterministic**: dùng IV cố định (hoặc suy ra từ chính plaintext, như mode SIV) → cùng plaintext luôn cho cùng ciphertext.

|                                                           | Random IV (chuẩn)            | Static IV (deterministic)                                  |
| --------------------------------------------------------- | ---------------------------- | ---------------------------------------------------------- |
| Leak equality giữa các bản ghi                            | ❌ Không                     | ⚠️ Có — 2 bản ghi trùng giá trị sẽ có ciphertext giống hệt |
| Equality match trên cột mã hóa (`WHERE col = encrypt(x)`) | ❌ Không thể                 | ✅ Hoạt động                                               |
| Idempotent khi ghi lại/re-sync                            | ❌ Mỗi lần một ciphertext    | ✅ Ổn định                                                 |
| Lưu trữ                                                   | Phải lưu IV cạnh mỗi giá trị | Không tốn thêm                                             |

Deterministic encryption là một chế độ **chính thống, có tên tuổi** trong ngành — AWS DynamoDB Encryption Client và MongoDB Client-Side Field Level Encryption đều cung cấp nó như một lựa chọn cho _field cần query_, kèm cảnh báo về leak equality. Với PII kiểu email/phone, cái leak "hai bản ghi này trùng email" (mà không biết email là gì) thường được chấp nhận.

### 3.5. 📌 Ví dụ thực tế — implement trong nollie-api

Toàn bộ nằm ở [src/services/encryption/encryption.service.ts](src/services/encryption/encryption.service.ts):

```typescript
private readonly algorithm = 'aes-256-cbc';

// Key: pad/cắt về đúng 32 bytes (256 bit), đọc từ env ENCRYPTION_KEY
const key = Buffer.from(encryptionKey.padEnd(32, '0').substring(0, 32));
// IV tĩnh: env ENCRYPTION_IV, 32 hex chars → 16 bytes, validate cứng
const iv = Buffer.from(encryptionIV, 'hex');

// Encrypt: utf8 → hex
const cipher = crypto.createCipheriv(this.algorithm, key, iv);
let encrypted = cipher.update(value, 'utf8', 'hex');
encrypted += cipher.final('hex');
```

Các quyết định thiết kế đáng học:

- **Output hex** thay vì base64: dễ nhận diện bằng regex, an toàn tuyệt đối khi lưu vào cột text.
- **Decrypt fail-safe**: `decryptValue` không bao giờ throw — giải mã lỗi thì trả về nguyên giá trị gốc. Lý do: DB còn lẫn dữ liệu cũ chưa mã hóa (ràng buộc 1.4-2); một bản ghi cũ không được phép làm crash cả list API.
- **Heuristic `isEncrypted`**: trước khi giải mã, đoán một giá trị có phải ciphertext không — hex thuần, ≥32 ký tự, độ dài chẵn (hệ quả PKCS#7 ở 3.2), không khớp pattern plaintext phổ biến (email `@.`, số điện thoại, tên chỉ chữ cái), entropy tối thiểu. Đây là cái giá của migrate dần.
- **Cache key/IV** parse từ env một lần duy nhất.

---

## 4. Search trên dữ liệu mã hóa — Blind Index

### 4.1. Vì sao không search trực tiếp được

`WHERE email LIKE 'john%'` trên cột ciphertext là vô nghĩa. Vì tính **diffusion** (3.1), plaintext bắt đầu bằng `"john"` **không** cho ra ciphertext bắt đầu bằng một chuỗi cố định nào — đổi ký tự thứ 5 của plaintext làm thay đổi toàn bộ block ciphertext. Deterministic encryption (3.4) chỉ cứu được **equality toàn chuỗi**, không cứu được prefix/substring.

### 4.2. Các hướng lý thuyết cho searchable encryption

1. **Deterministic encryption** — equality match. Cần thiết nhưng không đủ cho autocomplete.
2. **Order-preserving encryption** — cho range/sort nhưng leak thứ tự → các nghiên cứu đã khôi phục được plaintext từ hệ thống OPE thật. Loại.
3. **Searchable Symmetric Encryption (SSE)** học thuật — index mã hóa chuyên dụng, đẹp trên giấy, không có lib production-grade phổ biến. Loại với đa số stack.
4. **Homomorphic** — quá chậm. Loại.
5. **Blind index** ✅ — lưu thêm một cột **hash một chiều** của giá trị (hoặc các biến thể của nó). Search bằng cách hash từ khóa theo đúng cách đó rồi so _hash với hash_. Cột index "mù" vì từ hash không suy ngược ra giá trị. Đây là cách tiếp cận của thư viện **CipherSweet** (Paragon Initiative) và là lời giải phổ biến nhất trong thực tế.

### 4.3. Ý tưởng blind index + prefix tokens

Blind index cơ bản chỉ trả lời equality: "hash của từ khóa có bằng hash đã lưu không". Để có **prefix search** (`LIKE 'x%'`), mở rộng tự nhiên là: thay vì hash 1 giá trị, hash **mọi prefix của giá trị**.

Khi lưu giá trị `"Charles"`:

```
chuẩn hóa:   "charles"          (lowercase, trim, bỏ ký tự đặc thù như '+' đầu SĐT)
prefix ≥3:   ["cha", "char", "charl", "charle", "charles"]
hash:        mỗi token → SHA-256 → cắt ngắn (ví dụ 16 hex chars)
lưu:         cột tokens = ["9f86d081884c7d65", "1ba7...", ...]   (mảng, ví dụ JSONB)
```

Khi search `"char"`:

```
"char" → chuẩn hóa → prefix hóa → hash → ["h(cha)", "h(char)"]
WHERE tokens ⊇ {h(cha), h(char)}        -- containment: tokens của row chứa toàn bộ tokens của từ khóa
```

Row khớp ⟺ mảng token của row **chứa toàn bộ** token của từ khóa ⟺ giá trị gốc bắt đầu bằng từ khóa. Đúng ngữ nghĩa `LIKE 'char%'` mà DB không hề biết plaintext. Trên Postgres, phép containment này map thẳng vào toán tử JSONB `@>`.

### 4.4. Các tham số thiết kế và lý do

- **Độ dài token tối thiểu** (thường = 3): token 1–2 ký tự có không gian giá trị quá nhỏ (vài chục đến ~1300 khả năng) → dò ngược bằng brute-force/frequency analysis là tầm thường. Đồng thời search 1 ký tự trên bảng lớn cũng vô nghĩa về UX. Từ khóa ngắn hơn ngưỡng → **không search được, trả rỗng** — đây là hành vi đúng theo thiết kế.
- **Cắt ngắn hash** (ví dụ 16 hex = 64 bit): đủ để collision gần như không xảy ra ở quy mô hàng triệu token, tiết kiệm một nửa storage. Collision nếu có chỉ gây **false positive** (thừa row), không bao giờ false negative — chấp nhận được cho search UI.
- **Cap độ dài prefix** (ví dụ 50 ký tự): chặn giá trị bất thường sinh hàng trăm token.
- **Từ khóa không index được phải trả về `FALSE` (no match)** — nguyên tắc quan trọng nhất phía đọc, vì có hai cái bẫy đối xứng nhau mà hệ thống nào cũng dễ rơi vào:
  1. **Bẫy match-tất-cả**: nếu tokenizer trả mảng rỗng mà vẫn emit điều kiện containment, thì containment với tập rỗng (`tokens ⊇ ∅`) đúng với *mọi* row — query match cả bảng.
  2. **Bẫy fallback về ciphertext**: nếu "chữa" bằng cách cho từ khóa không index được quay về so sánh trực tiếp trên cột ciphertext, kết quả còn tệ hơn — ciphertext là hex/base64 nên một từ khóa ngắn khớp *tình cờ* với gần như mọi row (chuỗi hex nào chẳng chứa `"d"`), vừa sai kết quả vừa full-table scan. Câu trả lời trung thực cho "từ khóa không index được" là **no match**, không bao giờ là một phép so sánh thay thế trên ciphertext.
- **Quyết định trên tokens *sau* chuẩn hóa, không phải trên từ khóa thô**: kiểm tra `keyword.length >= 3` là chưa đủ — từ khóa toàn whitespace, hay `"+1"` (bỏ `+` xong còn 1 ký tự) vượt qua kiểm tra độ dài thô nhưng vẫn không sinh được token nào. Điều kiện đúng là `tokens.length > 0` trên kết quả cuối của tokenizer.

### 4.5. 📌 Ví dụ thực tế — phía ghi trong nollie-api

Mẫu thiết kế: một **repository base class** làm mã hóa trong suốt với service layer. Repository của model có PII kế thừa `BaseEncryptedRepository` ([src/database/base-encrypted.repository.ts](src/database/base-encrypted.repository.ts)) và khai báo field nào mã hóa/searchable:

```typescript
// src/modules/customers/repositories/customer.repository.ts
getEncryptedFields(): IEncryptedFields {
  return {
    firstName:    { encrypt: true, dataType: EFieldDataType.STRING, searchable: true },
    fullName:     { encrypt: true, dataType: EFieldDataType.STRING, searchable: true },
    primaryEmail: { encrypt: true, dataType: EFieldDataType.STRING, searchable: true },
    // ...
  };
}
```

Base repo tự động:

- **create/update**: mã hóa field `encrypt: true`; field `searchable: true` sinh thêm blind index ghi vào cột `{field}_tokens` (JSONB).
- **find**: giải mã kết quả trước khi trả về.
- Tất cả gate bằng flag env — deployment không cần mã hóa thì mọi thứ là pass-through.

Service layer gọi `repository.create/find` như bình thường, **không biết gì về mã hóa** — đây là điểm ăn tiền của mẫu này: một chỗ đúng thì mọi nơi đúng, không ai quên mã hóa một field.

Dữ liệu cũ backfill qua queue riêng ([src/services/bull/encryption.queue.ts](src/services/bull/encryption.queue.ts)) — mã hóa dần từng batch thay vì một migration khổng lồ khóa bảng.

### 4.6. 📌 Ví dụ thực tế — phía đọc: rewrite điều kiện WHERE

`applyBlindIndexCondition` ([base-encrypted.repository.ts:343](src/database/base-encrypted.repository.ts#L343)) chặn mọi điều kiện WHERE nhắm vào field mã hóa searchable và **thay thế tại chỗ** bằng điều kiện trên cột token:

```typescript
const tokens = this.encryptionService.generateBlindIndex(
  term.replace(/%/g, ''),
);

delete processed[fieldKey]; // bỏ điều kiện trên cột ciphertext
processed[blindIndexField] = tokens.length
  ? literal(`"${blindIndexField}"::jsonb @> '${JSON.stringify(tokens)}'::jsonb`)
  : literal('FALSE'); // không token hóa được → không match gì
```

Đoạn code này áp dụng đúng cả hai nguyên tắc cuối của 4.4: nhánh `FALSE` cho tokens rỗng, và quyết định trên `tokens.length` sau chuẩn hóa. Đáng nói là codebase này từng rơi vào đúng "bẫy fallback về ciphertext" mô tả ở 4.4 trước khi có nhánh `FALSE` — search 1 ký tự trả về gần như toàn bộ danh sách khách hàng — nên đây không phải rủi ro lý thuyết.

### 4.7. Giới hạn bảo mật của blind index

Blind index là **giảm leak, không phải zero leak** — cần biết rõ để không ngộ nhận:

1. **Hash không key vs có key**: nếu dùng hash thuần (SHA-256), kẻ có DB dump tự tính được `sha256("cha")` và dò ngược token ngắn/phổ biến (tên người, domain email phổ biến). Chuẩn tốt hơn là **HMAC-SHA256 với một key riêng** (khác key mã hóa) — khi đó không có key thì không tính được hash để dò. Đổi từ hash thuần sang HMAC yêu cầu re-index toàn bộ. _(nollie-api hiện dùng SHA-256 thuần cắt 16 hex — điểm nâng cấp tiềm năng.)_
2. **Leak prefix-equality giữa các row**: hai bản ghi chung chuỗi token đầu → biết chúng có giá trị cùng prefix.
3. **Số lượng token ≈ độ dài giá trị** → leak xấp xỉ độ dài plaintext.
4. **Chỉ prefix search** — không substring (`%abc%`), không fuzzy. Muốn substring phải sinh n-gram tokens, nổ số lượng token và nới rộng leak.

Threat model chấp nhận các leak này khi mục tiêu chính là _DB dump không đọc trực tiếp được PII_, không phải kháng một đối thủ có tài nguyên phân tích offline vô hạn.

---

## 5. Tổng kết

### 5.1. Kiến trúc tổng thể của lời giải

```
                      ghi (create/update)
Service ──► Encrypted Repository ──► AES-256-CBC (deterministic) ──► cột ciphertext
                      │              └► blind index (prefix ≥3 → hash cắt ngắn) ──► cột {field}_tokens
                      │
                      đọc (find + WHERE trên field mã hóa)
                      └──► rewrite: WHERE field=... ➜ WHERE tokens ⊇ tokens(từ khóa)
                           kết quả ──► decrypt (fail-safe) ──► plaintext cho service
```

### 5.2. Bản đồ quyết định & trade-off

| Quyết định          | Lựa chọn chuẩn                                        | Trade-off phải hiểu                                         |
| ------------------- | ----------------------------------------------------- | ----------------------------------------------------------- |
| Tầng mã hóa         | Application-level (chồng lên at-rest)                 | Code phức tạp hơn; key management thành trách nhiệm của app |
| Thuật toán          | AES-256                                               | — (mặc định ngành)                                          |
| Mode                | GCM cho hệ mới; CBC nếu cần deterministic             | CBC không có authenticated encryption                       |
| IV                  | Random per-record (chuẩn) hoặc static (deterministic) | Static leak equality — đổi lấy so khớp/idempotency          |
| Search              | Blind index prefix-token                              | Chỉ prefix; leak prefix-equality + độ dài                   |
| Hash cho index      | HMAC có key (tốt) / hash thuần (yếu hơn)              | Hash thuần dò ngược được token ngắn                         |
| Từ khóa dưới ngưỡng | `FALSE` — no match                                    | Không search được term ngắn — đúng chủ đích                 |
| Dữ liệu cũ lẫn lộn  | Heuristic nhận diện + decrypt fail-safe               | Heuristic có thể đoán sai ở ca hy hữu                       |
