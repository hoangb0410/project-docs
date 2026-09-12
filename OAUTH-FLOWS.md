# Đăng nhập OAuth với Google và Facebook

---

## 1. Bối cảnh

### Các bên tham gia

| Trong tài liệu này             | Theo spec OAuth 2.0                     | Vai trò                                                        |
| ------------------------------ | --------------------------------------- | -------------------------------------------------------------- |
| **User**                       | Resource Owner                          | Người sở hữu tài khoản, là người bấm đồng ý                     |
| **Browser**                    | User-Agent                              | Nơi user bấm nút và nơi user thực sự đồng ý cấp quyền           |
| **Backend của mình**           | Client                                  | Nơi cần biết chắc "người này là ai" để tạo session              |
| **Provider** (Google/Facebook) | Authorization Server + Resource Server  | Bên duy nhất biết user là ai và chứng thực được điều đó         |

Authorization Server và Resource Server thường là hai host khác nhau dù cùng một provider — `accounts.google.com` cấp token, `www.googleapis.com` nhận token. Phân biệt này quan trọng khi đọc tài liệu của provider.

**Client không đồng nghĩa với backend.** Client là bên đăng ký `client_id` với provider và gọi `/token`. Thành phần nào của mình làm việc đó thì là Client; phần còn lại chỉ là User-Agent chở redirect. Cùng một FE, ở token flow nó là Client, ở authorization code nó chỉ là User-Agent.

| Luồng                       | Client                                   | User-Agent                    | Backend nếu không phải Client                              |
| --------------------------- | ---------------------------------------- | ----------------------------- | ---------------------------------------------------------- |
| Token flow                  | **FE** — SDK trong browser nhận token     | Chính nó                      | Nhận token từ FE rồi verify — vai trò gần Resource Server  |
| Authorization code          | **BE**                                   | Browser                       | —                                                          |
| PKCE, public client         | **SPA / mobile app**                     | Browser tab / in-app browser  | Không tham gia                                             |
| PKCE, backend (ResDiary)    | **BE**                                   | Browser                       | —                                                          |

### Bước 0: đăng ký app với provider

Mọi luồng đều bắt đầu **trước** sequence — app phải được đăng ký ở console của provider (Google Cloud Console, Meta for Developers, Square Developer Dashboard, ResDiary partner portal). Không có bước này thì không có `client_id`, và `/authorize` không biết user đang uỷ quyền cho ai.

| Khai báo       | Dùng để                                                                     |
| -------------- | --------------------------------------------------------------------------- |
| Tên, logo      | Hiện trên màn hình consent                                                  |
| Loại client    | Quyết định có được cấp `client_secret` không, có bắt PKCE không              |
| `redirect_uri` | Provider chỉ trả `code` về đúng URI này                                      |
| Scope          | Nhiều provider bắt review từng scope trước khi cho app production dùng       |

Nhận về `client_id` (công khai) và, chỉ với confidential client, `client_secret`. Trong nollie-api chúng nằm ở env: `CLIENT_ID` / `CLIENT_SECRET` (Square), `CLIENT_ID_RES` / `CLIENT_SECRET_RES` / `REDIRECT_URI_RES` (ResDiary).

User uỷ quyền cho **`client_id`**, không phải cho server hay domain — đổi `client_id` là mọi token đã cấp vô hiệu, user phải connect lại.

### Bài toán

Backend cần một **bằng chứng danh tính đáng tin từ provider**, rồi từ đó tạo session của riêng mình (ở project này là cookie `access_token` / `refresh_token`).

Backend **không được tin** thông tin do browser tự khai. Browser nằm trong tay user, ai cũng sửa được request. Bằng chứng phải là thứ backend tự xác minh được với provider.

Ba luồng dưới đây khác nhau ở đúng hai chỗ: **bằng chứng đi tới backend bằng đường nào**, và **ai giữ bí mật để đổi được nó**.

### Loại client quyết định luồng

| Loại client                             | Giữ được `client_secret`? | Luồng dùng được                                        |
| --------------------------------------- | ------------------------- | ------------------------------------------------------ |
| Confidential — backend, server-side web | Có                        | Token flow, Authorization code — PKCE tuỳ chọn, chồng thêm lên secret |
| Public — SPA, mobile, desktop, CLI      | Không                     | Authorization code + PKCE — bắt buộc, PKCE thay secret  |

---

## 2. Token flow

### Ý tưởng

Provider phát token trực tiếp cho **browser**. Browser gửi token đó lên backend. Backend mang token đi hỏi provider "token này có thật không, có phải phát cho app của tôi không".

Bằng chứng ở đây là **token của provider**, và nó đi xuyên qua browser.

### Luồng

```mermaid
%%{init: {
  'sequence': { 'messageAlign': 'left', 'width': 200 },
  'themeVariables': {
    'noteBkgColor': '#1F2937',
    'noteTextColor': '#F9FAFB',
    'noteBorderColor': '#111827'
  }
}}%%
sequenceDiagram
    participant B as Browser
    participant A as Our API
    participant P as Google / Facebook

    B->>P: 1. SDK popup — user consents
    P->>B: 2. ID token / access token
    Note over B,P: token của provider nằm trong browser
    B->>A: 3. POST /auth/google { token }
    A->>P: 4. verify — JWKS / debug_token
    P->>A: 5. sub, email, email_verified
    Note over A: 6. find or create user + link
    A->>B: 7. Set-Cookie: access_token, refresh_token
```

1. User bấm nút. SDK của provider (Google Identity Services / Facebook JS SDK) mở popup.
2. User đồng ý. Provider trả token về cho JS trong browser:
   - **Google** trả `id_token` — JWT có chữ ký, chứa sẵn `sub`, `email`, `email_verified`
   - **Facebook** trả `access_token` — chuỗi đục, không đọc được nội dung, phải hỏi lại Facebook
3. Browser POST token lên backend.
4. Backend xác minh:
   - **Google**: verify chữ ký JWT bằng public key của Google (thư viện cache key nên gần như không cần gọi mạng), check `aud` đúng `client_id` của mình
   - **Facebook**: gọi `GET /debug_token` để hỏi token có hợp lệ không, rồi `GET /me` để lấy profile
5. Provider trả về danh tính.
6. Backend tìm hoặc tạo user, gắn liên kết provider.
7. Backend set cookie session của mình.

### Chỗ dễ sai nhất

Ở bước 4, nếu chỉ kiểm "token này có hợp lệ với provider không" mà **không** kiểm "token này có phát cho app của tôi không" thì hổng nghiêm trọng: kẻ tấn công lấy token từ một app Google/Facebook bất kỳ mà họ kiểm soát rồi gửi lên, và login được vào hệ thống của bạn.

| Provider | Phải check                                        |
| -------- | ------------------------------------------------- |
| Google   | `aud` == `client_id` của mình                     |
| Facebook | `debug_token` trả về `app_id` == app id của mình  |

### Không phải implicit grant

Luồng này **không phải** _implicit grant_ — loại đã bị OAuth 2.1 khai tử. Google Identity Services trả ID token và Facebook JS SDK trả access token đều là luồng chính thức hai bên khuyến nghị cho mục đích **xác thực** (biết user là ai). Implicit grant là chuyện khác: nó trả _access token để gọi API_ qua URL fragment, và đó mới là thứ bị loại bỏ.

---

## 3. Authorization code flow

### Ý tưởng

Provider **không** phát token cho browser. Nó chỉ phát một `code` dùng một lần. Backend mang `code` đó cùng `client_secret` đi đổi lấy token, trong một request server-to-server mà browser không tham gia.

Bằng chứng vẫn là token, nhưng token **chưa bao giờ đi qua browser**.

### Luồng

```mermaid
%%{init: {
  'sequence': { 'messageAlign': 'left', 'width': 200 },
  'themeVariables': {
    'noteBkgColor': '#1F2937',
    'noteTextColor': '#F9FAFB',
    'noteBorderColor': '#111827'
  }
}}%%
sequenceDiagram
    autonumber
    actor U as User
    participant B as Browser
    participant A as Our API
    participant AS as Authorization Server
    participant RS as Resource Server

    U->>B: bấm "Sign in with Google"
    B->>A: GET /auth/google/redirect
    Note over A: sinh state random, lưu Redis, TTL ngắn
    A-->>B: 302 → {AS}/authorize?response_type=code<br/>&client_id&redirect_uri&scope&state
    B->>AS: GET /authorize
    AS->>U: đăng nhập + consent — trên domain của provider
    U->>AS: đồng ý
    Note over AS: sinh code — one-time, TTL ~30–60s
    AS-->>B: 302 → {redirect_uri}?code&state
    B->>A: GET /callback?code&state
    Note over A: so state với Redis, xoá ngay sau khi dùng
    alt state không khớp / hết hạn
        A-->>B: 400 — dừng flow
    end
    rect rgba(59,130,246,0.15)
        Note over A,AS: back-channel — browser không tham gia
        A->>AS: POST /token<br/>grant_type=authorization_code, code,<br/>redirect_uri, client_id, client_secret
        Note over AS: code chưa dùng? đúng client? đúng redirect_uri?
        AS-->>A: access_token, id_token,<br/>refresh_token nếu access_type=offline
    end
    Note over A: verify id_token → find or create user + link
    A-->>B: Set-Cookie + 302 → app
    opt gọi API provider thay mặt user
        A->>RS: GET /resource — Bearer access_token
        RS-->>A: 200
    end
```

1. User bấm một link thường — không cần JS, không cần SDK.
2. Browser gọi vào backend. Backend sinh `state` random, lưu server-side kèm TTL ngắn.
3. Backend redirect browser sang `/authorize` của provider, đính kèm `client_id`, `redirect_uri`, `scope`, `state`.
4. Browser mở trang provider.
5. User thấy trang đăng nhập và đồng ý **của chính Google/Facebook**, trên domain của họ.
6. User đồng ý. Provider sinh `code` dùng một lần, TTL rất ngắn.
7. Provider redirect browser về `redirect_uri` đã khai báo trước, đính kèm `code` và `state`.
8. Browser tự động gọi vào callback của backend. Backend so `state` với cái đã lưu, xoá ngay sau khi dùng.
9. `state` không khớp hoặc hết hạn → dừng, không đổi `code`.
10. Backend gọi thẳng token endpoint, gửi `code` + `client_id` + `client_secret` + `redirect_uri`. **Request server-to-server, browser không thấy gì.** Provider kiểm tra `code` chưa dùng, đúng client, đúng `redirect_uri`.
11. Provider trả `access_token` và `id_token`. Xin `access_type=offline` (Google) thì có thêm `refresh_token`.
12. Backend verify `id_token`, tìm hoặc tạo user, gắn liên kết provider — **giống hệt bước 6 của token flow** — rồi set cookie, redirect về frontend.
13. – 14. Chỉ khi cần gọi API provider thay mặt user: backend dùng `access_token` gọi Resource Server.

### Ba cơ chế bảo vệ

| Cơ chế          | Chứng minh / chống gì                                                                                   | Yêu cầu                                                        |
| --------------- | ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------- |
| `client_secret` | "Tôi đúng là app đó" — chỉ backend đổi được `code` lấy token                                             | Chỉ nằm ở backend, không bao giờ ra browser                     |
| `state`         | CSRF — kẻ tấn công dụ victim gọi callback với `code` **của kẻ tấn công**, victim đăng nhập nhầm vào tài khoản của họ | Sinh và lưu server-side, dùng một lần, có TTL       |
| `redirect_uri`  | Kẻ tấn công không đổi hướng `code` sang server của họ được                                               | Khai báo chính xác ở console provider, provider so khớp tuyệt đối |

---

## 4. Authorization code + PKCE

### Ý tưởng

PKCE (Proof Key for Code Exchange, đọc là "pixy") sinh ra để giải bài toán: **làm sao dùng authorization code flow khi không thể giữ `client_secret` bí mật?** — tình huống của **public client** (SPA, mobile, desktop), nơi code nằm trong tay user, nhúng secret vào là coi như công khai.

Nhưng cơ chế không gắn với loại client. Bất kỳ client nào cũng dùng được, và confidential client dùng **kèm** `client_secret` thì được thêm một lớp — OAuth 2.1 khuyến nghị PKCE cho mọi client.

PKCE bổ sung cho `client_secret` — bí mật **cố định, dùng mãi** — một cặp bí mật **sinh mới mỗi lần đăng nhập**:

| Giá trị          | Cách sinh                            | Ai thấy                                                                  |
| ---------------- | ------------------------------------ | ------------------------------------------------------------------------ |
| `code_verifier`  | Chuỗi random 43–128 ký tự            | Chỉ client — gửi duy nhất ở bước đổi token                                |
| `code_challenge` | `BASE64URL(SHA256(code_verifier))`   | Công khai — đi qua URL `/authorize`                                       |

Ai sinh và giữ `code_verifier` tuỳ client là gì:

| Client                    | Sinh verifier ở đâu | Giữ ở đâu                                    | Lúc đổi code gửi                        |
| ------------------------- | ------------------- | -------------------------------------------- | --------------------------------------- |
| SPA                       | Trong browser       | Memory / session storage                     | `code_verifier`                         |
| Mobile / desktop          | Trong app           | Memory                                       | `code_verifier`                         |
| Backend (confidential)    | Trên server         | Server-side, key theo `state` (Redis, TTL ngắn) | `code_verifier` **+** `client_secret` |

Ví dụ ngay trong nollie-api: kết nối ResDiary — backend sinh verifier, lưu Redis `res-diary:code_verifier:{state}` TTL 5 phút, callback đọc lại verifier theo `state`, gọi `/oauth/token` với Basic auth `client_id:client_secret` **và** `code_verifier`.

### Luồng

```mermaid
%%{init: {
  'sequence': { 'messageAlign': 'left', 'width': 200 },
  'themeVariables': {
    'noteBkgColor': '#1F2937',
    'noteTextColor': '#F9FAFB',
    'noteBorderColor': '#111827'
  }
}}%%
sequenceDiagram
    autonumber
    actor U as User
    participant C as Client<br/>(SPA / mobile / backend)
    participant B as User-Agent<br/>(browser tab / in-app browser)
    participant AS as Authorization Server
    participant RS as Resource Server

    U->>C: bấm "Sign in" / "Connect"
    rect rgba(34,197,94,0.15)
        Note over C: code_verifier = random 43–128 ký tự
        Note over C: code_challenge = BASE64URL(SHA256(code_verifier))
        Note over C: giữ code_verifier + state —<br/>app: memory / session storage<br/>backend: Redis key theo state, TTL ngắn
    end
    C->>B: mở {AS}/authorize?response_type=code&client_id<br/>&redirect_uri&scope&state<br/>&code_challenge&code_challenge_method=S256
    B->>AS: GET /authorize
    Note over AS: lưu code_challenge gắn với phiên authorize này
    AS->>U: đăng nhập + consent
    U->>AS: đồng ý
    Note over AS: sinh code, gắn với code_challenge đã lưu
    AS-->>B: 302 → {redirect_uri}?code&state
    B->>C: callback / deep link — code, state
    Note over C: so state, lấy lại code_verifier theo state
    rect rgba(34,197,94,0.15)
        Note over C,AS: đổi code — code_verifier thay hoặc kèm client_secret
        C->>AS: POST /token<br/>grant_type=authorization_code, code,<br/>redirect_uri, client_id, code_verifier<br/>[+ client_secret nếu là confidential client]
        Note over AS: BASE64URL(SHA256(code_verifier)) == code_challenge đã lưu?
        alt không khớp — code bị chặn, kẻ tấn công không có verifier
            AS-->>C: 400 invalid_grant
        end
        AS-->>C: access_token, id_token, refresh_token
    end
    Note over C: xoá code_verifier — dùng một lần
    C->>RS: GET /resource — Bearer access_token
    RS-->>C: 200
```

1. User bấm nút. Client sinh `code_verifier` random, hash ra `code_challenge`, giữ verifier và `state` — app giữ trong memory, backend lưu server-side key theo `state`. Verifier **không** gửi đi ở bước này.
2. Client mở `/authorize` của provider qua trình duyệt — SPA/backend redirect tab hiện tại, mobile dùng in-app browser (`ASWebAuthenticationSession` / Custom Tabs) — kèm `code_challenge` và `code_challenge_method=S256`.
3. Provider nhận request, lưu `code_challenge` gắn với phiên authorize này.
4. User thấy trang đăng nhập và đồng ý của provider.
5. User đồng ý. Provider sinh `code`, gắn với `code_challenge` đã lưu.
6. Provider redirect về `redirect_uri` kèm `code` và `state`.
7. Trình duyệt chuyển `code` + `state` về client — redirect với SPA, deep link / custom scheme với mobile, gọi thẳng endpoint callback với backend. Client so `state` và lấy lại `code_verifier` tương ứng.
8. Client đổi `code`, lần này gửi `code_verifier` **gốc** — thứ chưa từng xuất hiện trên đường truyền. Public client không có secret để gửi; confidential client gửi kèm `client_secret` (Google cho chồng cả hai; Facebook chọn một trong hai — xem bảng dưới).
9. Provider hash verifier, so với challenge đã lưu. Không khớp → `invalid_grant`, `code` bị huỷ.
10. Khớp → cấp token. Client xoá `code_verifier`, không dùng lại.
11. – 12. Client dùng `access_token` gọi Resource Server.

### Vì sao thay được `client_secret`

Kẻ tấn công chặn được `code` ở bước 6–7 (qua log, deep link bị đăng ký trùng trên mobile, referrer) vẫn **không đổi được token**, vì không có `code_verifier` — nó chưa bao giờ rời khỏi app.

Khác biệt cốt lõi: secret bị lộ một lần là lộ vĩnh viễn. Verifier chỉ dùng cho **một** lần login, lộ cũng vô dụng.

| `code_challenge_method` | Ý nghĩa                              | Dùng không?                                                        |
| ----------------------- | ------------------------------------ | ------------------------------------------------------------------ |
| `S256`                  | `challenge = BASE64URL(SHA256(verifier))` | Luôn                                                          |
| `plain`                 | `challenge = verifier`               | Không — verifier lộ ngay ở bước 2, chỉ dành cho nền tảng không hash được |

### PKCE không thay thế `state`

Có PKCE rồi **vẫn phải** giữ `state`. Hai cơ chế chống hai hướng tấn công ngược nhau:

| Cơ chế  | Chống gì                                                                          |
| ------- | --------------------------------------------------------------------------------- |
| `state` | CSRF — kẻ tấn công ép nạn nhân dùng `code` **của kẻ tấn công**                      |
| PKCE    | Code interception — kẻ tấn công chiếm `code` **của nạn nhân** để đổi lấy token       |

### Hỗ trợ thực tế ở Google và Facebook

Cả hai đều hỗ trợ, nhưng định vị khác nhau:

|                             | Google                                                                                                                                                                                                                     | Facebook                                                                                                                              |
| --------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| Hỗ trợ                      | Có — [discovery document](https://accounts.google.com/.well-known/openid-configuration) khai `code_challenge_methods_supported: ["plain", "S256"]`                                                                         | Có — qua [OIDC Code Flow with PKCE](https://developers.facebook.com/documentation/facebook-login/guides/advanced/oidc-token)          |
| Document ở đâu              | Trang [iOS & Desktop Apps](https://developers.google.com/identity/protocols/oauth2/native-app). Trang [Web Server Apps](https://developers.google.com/identity/protocols/oauth2/web-server) **không** liệt kê tham số PKCE | Trang OIDC riêng, không có trong [Manual Login Flow](https://developers.facebook.com/docs/facebook-login/guides/advanced/manual-flow) |
| Quan hệ với `client_secret` | Cộng thêm — dùng được **cùng** secret (defense in depth)                                                                                                                                                                   | **Hoặc cái này hoặc cái kia** — gửi `client_secret` **hoặc** `code_verifier`; bỏ secret thì verifier thành bắt buộc                   |

Với Facebook, PKCE là **phương án thay thế** cho `client_secret`, đúng tinh thần gốc. Với Google thì chồng cả hai được.

**PKCE không bị giới hạn cho SPA/mobile.** Cả hai provider đều nhận nó ở luồng authorization code bất kể loại client. Nó _trông_ như chỉ dành cho SPA/mobile vì tài liệu PKCE nằm ở trang riêng cho các loại app đó — nơi PKCE **bắt buộc**. Web app có backend thì PKCE là **tuỳ chọn**.

> Chưa xác minh: Google khai hỗ trợ PKCE ở **cấp authorization server** qua discovery document. OAuth client loại "Web application" có nhận `code_challenge` hay không thì Google không document. Các thư viện phổ biến (Auth.js, Spring Security) vẫn gửi PKCE cho web client và chạy được, nhưng nếu định dựa vào thì nên test thật một lần.

### Khi nào project này cần tới nó

Với đăng nhập Google/Facebook hiện tại **không bắt buộc** — backend giữ được `client_secret`, authorization code thường đã đủ. Nhưng đã có chỗ dùng: kết nối ResDiary chạy PKCE từ backend.

| Tình huống                                              | Cần PKCE?                                   |
| ------------------------------------------------------- | ------------------------------------------- |
| Mobile app gọi trực tiếp provider, không qua backend     | Bắt buộc                                    |
| SPA thuần, không backend nào giữ secret                  | Bắt buộc                                    |
| Provider yêu cầu hoặc khuyến nghị PKCE (ResDiary)        | Làm theo provider, dù backend là confidential |
| Siết thêm một lớp cho luồng backend hiện có              | Tuỳ chọn — Google cho chồng, Facebook thì chọn một |

---

## 5. Refresh token

Áp dụng cho authorization code và PKCE — token flow không có refresh token của provider (xem §6).

```mermaid
%%{init: {
  'sequence': { 'messageAlign': 'left', 'width': 200 },
  'themeVariables': {
    'noteBkgColor': '#1F2937',
    'noteTextColor': '#F9FAFB',
    'noteBorderColor': '#111827'
  }
}}%%
sequenceDiagram
    participant A as Our API
    participant AS as Authorization Server
    participant RS as Resource Server

    A->>RS: 1. GET /resource — Bearer access_token
    RS->>A: 2. 401 invalid_token (hết hạn)
    A->>AS: 3. POST /token — grant_type=refresh_token
    Note over A,AS: PKCE không áp dụng ở đây —<br/>code_verifier chỉ dùng cho authorization_code
    AS->>A: 4. access_token mới (+ refresh_token mới nếu rotation)
    Note over A: 5. lưu đè token cũ
    A->>RS: 6. retry request
    RS->>A: 7. 200
```

| Điểm hay bỏ sót           | Chi tiết                                                                                                                                                            |
| ------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rotation                  | Nhiều AS trả `refresh_token` **mới** mỗi lần và thu hồi cái cũ. Lưu đè. Nếu AS thấy một refresh token đã dùng bị dùng lại, nó thường thu hồi cả chuỗi — coi là dấu hiệu bị đánh cắp |
| `invalid_grant` là trạng thái cuối | Refresh token bị revoke, hết hạn, user đổi mật khẩu đều ra lỗi này. Retry vô ích — đánh dấu kết nối hỏng, bắt user authorize lại. Chỉ retry với 5xx / lỗi mạng |
| Refresh chủ động          | Cron/scheduler refresh trước khi hết hạn tốt hơn đợi 401 rồi retry — lỗi không lộ ra người dùng                                                                       |

---

## 6. So sánh

### Bảng đối chiếu

|                                  | Token flow                                                                                                  | Authorization code                                              | Auth code + PKCE                                              |
| -------------------------------- | ----------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------- | ------------------------------------------------------------- |
| Token provider trong browser     | Có — XSS lấy được                                                                                           | Không, chỉ `code` dùng 1 lần — XSS không lấy được               | Public client: có, app chính là client. Backend client: không |
| Bí mật để đổi / verify token     | Không có — backend tự verify `aud` / `app_id`                                                               | `client_secret` cố định                                         | `code_verifier` sinh mới mỗi lần login, confidential client kèm thêm `client_secret` |
| Cần `client_secret`              | Google: không. Facebook: cho `debug_token`                                                                  | Bắt buộc cả hai                                                 | Public: không. Confidential: gửi kèm                          |
| SDK provider trong browser       | Bắt buộc                                                                                                    | Không                                                           | Không                                                         |
| Bị ad blocker chặn               | Có — `connect.facebook.net` nằm trong hầu hết blocklist, SDK không load thì nút **im lặng không làm gì**    | Không                                                           | Không                                                         |
| JS bên thứ ba / CSP              | CSP phải lỏng hơn, thêm bề mặt tracking                                                                     | Không load gì từ Google/Meta, CSP siết được                     | Như authorization code                                        |
| Redirect                         | Không — popup, SPA không mất state                                                                          | Full-page, 2 vòng — SPA phải tự lưu/phục hồi state              | Full-page, 2 vòng                                             |
| Chống CSRF                       | SDK tự xử lý                                                                                                | Tự làm: `state`                                                 | Tự làm: `state`                                               |
| Refresh token của provider       | Không. Google chỉ trả `id_token`, không gọi được API. Facebook đổi được sang [long-lived ~60 ngày](https://developers.facebook.com/docs/facebook-login/guides/access-tokens/get-long-lived) qua `fb_exchange_token` (cần app secret), hết hạn phải login lại | Có, với `access_type=offline` — gọi được API provider về sau | Có                                     |
| Endpoint backend cần thêm        | 0                                                                                                           | +2 mỗi provider                                                 | App gọi thẳng provider: 0. Backend làm client: +2 như authorization code |
| Chỗ dễ làm sai                   | Quên verify `aud` / `app_id` — hổng nghiêm trọng, không có gì nhắc                                          | `state` làm sai thành lỗ bảo mật; `redirect_uri` phải khai cho **từng** môi trường, preview deploy domain động khá mệt | Verifier không đủ random, mất qua vòng redirect, hash sai; Facebook phải chuyển sang luồng OIDC riêng; tài liệu cả hai provider nằm lệch trang chính |
| Vị thế trong chuẩn               | Luồng xác thực chính thức của hai provider, không phải implicit grant                                        | Chuẩn chung, dùng lại được cho mobile/native                    | OAuth 2.1 khuyến nghị mặc định cho public client              |
| **Cần thiết** cho                | Web có backend                                                                                              | Web có backend                                                  | Bắt buộc cho SPA / mobile không giữ được secret; tuỳ chọn cho backend, hoặc khi provider yêu cầu |

### Chọn cái nào

Câu hỏi quyết định không phải "cái nào an toàn hơn" mà là "có chỗ nào an toàn để giữ `client_secret` không". Có thì dùng secret, không thì dùng PKCE.

| Tình huống                                                        | Chọn                        |
| ----------------------------------------------------------------- | --------------------------- |
| Chỉ cần đăng nhập, muốn UX popup gọn, không gọi API provider về sau | Token flow                  |
| Cần gọi API provider thay mặt user (cần refresh token)             | Authorization code          |
| Không muốn nhúng JS bên thứ ba, hoặc cần CSP chặt                  | Authorization code          |
| Lo ad blocker chặn Facebook SDK                                    | Authorization code          |
| Không có backend nào giữ được secret — mobile, SPA thuần           | Authorization code + PKCE   |

**Các luồng không loại trừ nhau.** Phần xử lý sau khi có danh tính giống nhau hoàn toàn, nên thêm luồng thứ hai về sau không phải viết lại — ví dụ web dùng authorization code, mobile dùng PKCE, chung một backend.

> Project này hiện đang dùng **token flow**. Contract cho frontend nằm ở [frontend-social-login-integration.md](./frontend-social-login-integration.md).

---

## 7. Setup nếu triển khai authorization code

### Google

- _Clients → web client → Authorized redirect URIs_: thêm `http://localhost:3000/api/auth/google/callback` và bản production.
- Client secret giờ **bắt buộc** trong env backend.

```
authorize  https://accounts.google.com/o/oauth2/v2/auth
token      https://oauth2.googleapis.com/token
```

### Facebook

- _Trường hợp sử dụng → Tùy chỉnh → Cài đặt → URI chuyển hướng OAuth hợp lệ_: thêm callback URL.
- App secret đã có. Toggle "Đăng nhập bằng SDK JavaScript" không cần nữa.

```
dialog  https://www.facebook.com/v26.0/dialog/oauth
token   https://graph.facebook.com/v26.0/oauth/access_token
```

> `v26.0` lấy từ URL dialog quan sát được (`facebook.com/v26.0/dialog/oauth`) — version SDK bên FE đang dùng. Vẫn nên đối chiếu _Cài đặt → Nâng cao → Nâng cấp phiên bản API_ để đặt `FACEBOOK_GRAPH_VERSION` cho đúng.

### Backend

Hai endpoint mỗi provider:

```
GET /api/auth/google/redirect
  sinh state random
  lưu Redis, TTL ngắn
  302 tới authorize URL

GET /api/auth/google/callback?code&state
  verify state với Redis, xoá ngay sau khi dùng
  POST token endpoint kèm code + client_id + client_secret + redirect_uri
  verify id_token trả về
  → linkAndIssueTokens(...)     // không đổi
  302 về frontend
```

Toàn bộ phần dưới **dùng lại nguyên vẹn**: bảng `user_social_accounts`, logic find-or-create, nhánh email + OTP cho provider không trả email verified, xử lý cookie.

### Frontend

SDK, callback, và bước POST token đều biến mất:

```html
<a href="/api/auth/google/redirect">Sign in with Google</a>
```

Hai màn email + OTP **vẫn phải giữ**: đổi luồng không làm Facebook trả email nếu app chưa qua App Review. Chỉ khác là API mang state đó qua redirect về app thay vì trả trong JSON response.

---

## 8. Tham chiếu tham số

Dùng chung cho mọi provider. Cột **Auth code** là luồng dùng `client_secret`, cột **+ PKCE** là luồng có `code_verifier`.

### `GET /authorize`

| Tham số                 | Auth code  | + PKCE     | Mô tả                                         |
| ----------------------- | ---------- | ---------- | --------------------------------------------- |
| `response_type`         | ✅         | ✅         | Luôn là `code`                                 |
| `client_id`             | ✅         | ✅         | Định danh client do AS cấp                     |
| `redirect_uri`          | ✅         | ✅         | Phải khớp chính xác URI đã đăng ký             |
| `scope`                 | Tuỳ AS     | Tuỳ AS     | Danh sách quyền, phân tách bằng space          |
| `state`                 | ✅ thực tế | ✅ thực tế | Chuỗi random chống CSRF, verify khi callback   |
| `code_challenge`        | —          | ✅         | `BASE64URL(SHA256(code_verifier))`             |
| `code_challenge_method` | —          | ✅         | `S256`                                         |

### `POST /token` — `grant_type=authorization_code`

`Content-Type: application/x-www-form-urlencoded`

| Tham số         | Auth code | + PKCE   | Mô tả                                                          |
| --------------- | --------- | -------- | -------------------------------------------------------------- |
| `grant_type`    | ✅        | ✅       | `authorization_code`                                            |
| `code`          | ✅        | ✅       | Code nhận ở callback — one-time, TTL ngắn                        |
| `redirect_uri`  | ✅        | ✅       | Phải trùng giá trị đã gửi ở `/authorize`                         |
| `client_id`     | ✅        | ✅       |                                                                  |
| `client_secret` | ✅        | Tuỳ AS   | Public client không có. Confidential client dùng PKCE thì tuỳ AS chấp nhận cả hai hay thay thế nhau |
| `code_verifier` | —         | ✅       | Chuỗi gốc, chưa từng xuất hiện trên đường truyền trước bước này   |

### `POST /token` — `grant_type=refresh_token`

| Tham số         | Auth code | + PKCE   | Mô tả                                                  |
| --------------- | --------- | -------- | ------------------------------------------------------ |
| `grant_type`    | ✅        | ✅       | `refresh_token`                                         |
| `refresh_token` | ✅        | ✅       | Token nhận ở lần đổi code trước                         |
| `client_id`     | ✅        | ✅       |                                                         |
| `client_secret` | ✅        | Tuỳ AS   | Như trên                                                |
| `scope`         | Tuỳ chọn  | Tuỳ chọn | Chỉ để **thu hẹp** scope, không mở rộng được            |
| `code_verifier` | —         | —        | Không dùng — PKCE chỉ áp dụng cho `authorization_code`  |

### Vị trí của client credentials

| Cách gửi                                               | Ghi chú                                            |
| ------------------------------------------------------ | -------------------------------------------------- |
| `Authorization: Basic BASE64(client_id:client_secret)` | Cách spec khuyến nghị                               |
| `client_id` + `client_secret` trong form body          | Nhiều AS chấp nhận, một số chỉ chấp nhận cách này   |

Đọc tài liệu của từng provider, đừng giả định.

### Response

| Field           | Luôn có | Mô tả                                               |
| --------------- | ------- | --------------------------------------------------- |
| `access_token`  | ✅      | Token gọi Resource Server                            |
| `token_type`    | ✅      | Thường là `Bearer`                                   |
| `expires_in`    | Tuỳ AS  | Số giây còn hiệu lực                                 |
| `refresh_token` | Tuỳ AS  | Chỉ có nếu xin đúng scope / tham số offline          |
| `scope`         | Tuỳ AS  | Scope thực sự được cấp, có thể hẹp hơn scope đã xin  |

```json
{
  "access_token": "2YotnFZFEjr1zCsicMWpAA",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "tGzv3JOkF0XG5Qx2TlKWIA",
  "scope": "read write"
}
```

### Lỗi thường gặp

| Error                   | Nguyên nhân điển hình                                                  |
| ----------------------- | ---------------------------------------------------------------------- |
| `invalid_grant`         | Code đã dùng / hết hạn, `code_verifier` sai, refresh token bị thu hồi   |
| `invalid_client`        | Sai `client_id` / `client_secret`, hoặc sai cách gửi credentials        |
| `redirect_uri_mismatch` | `redirect_uri` khác lúc đăng ký, hoặc khác lúc gọi `/authorize`         |
| `invalid_scope`         | Scope không tồn tại hoặc client chưa được cấp                           |
| `access_denied`         | User bấm từ chối ở màn hình consent                                     |

---

## 9. Spec tham chiếu

| Spec              | Nội dung                                                          |
| ----------------- | ----------------------------------------------------------------- |
| RFC 6749          | OAuth 2.0 Authorization Framework                                 |
| RFC 7636          | PKCE                                                              |
| RFC 6819          | OAuth 2.0 Threat Model and Security Considerations                |
| RFC 9700          | Best Current Practice for OAuth 2.0 Security                      |
| OAuth 2.1 (draft) | Gộp PKCE thành bắt buộc, loại bỏ implicit grant và password grant  |
