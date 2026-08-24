# Roadmap Frontend/React & Bộ câu hỏi phỏng vấn dành cho Backend Node.js Dev

> Dành cho bạn đã vững Backend Node.js, giờ mở rộng sang Fullstack (React). Tài liệu gồm 2 phần: **(1) Lộ trình học** và **(2) Bộ câu hỏi phỏng vấn FE kèm đáp án ngắn gọn**.

---

## Phần 1: Lộ trình học (Roadmap)

### 1. Nền tảng Web bắt buộc (nếu đã quên)
- HTML semantic (section, article, nav, form, accessibility cơ bản - aria, alt, label)
- CSS: box model, flexbox, grid, position, specificity, responsive (media query), units (rem/em/vh/%)
- CSS methodology: BEM, hoặc utility-first (Tailwind)

### 2. JavaScript nâng cao cho FE (khác với mindset BE)
- Event loop trong **browser** (khác Node một chút: microtask/macrotask, requestAnimationFrame)
- DOM API: querySelector, event delegation, event bubbling/capturing
- Closures, this, prototype (bạn chắc đã biết từ Node nhưng cần liên hệ context browser)
- Fetch API, AbortController, CORS (bạn hiểu CORS từ phía server rồi, giờ hiểu thêm từ phía client)
- ES modules trong browser (khác CommonJS bạn quen ở Node)

### 3. React Core
- JSX, Component (function component + hooks, không cần học class component sâu, chỉ cần đọc hiểu)
- Props, State, Lifting state up
- Hooks: useState, useEffect, useRef, useMemo, useCallback, useContext, useReducer
- Custom hooks
- Rendering behavior: re-render khi nào, key trong list, reconciliation/virtual DOM cơ bản
- Controlled vs Uncontrolled component (form)

### 4. State Management
- Local state vs Global state khi nào dùng
- Context API (dùng cho state đơn giản/ theme/auth)
- Thư viện phổ biến: Redux Toolkit, Zustand, Jotai (chọn 1-2 để học sâu, biết sơ các cái còn lại)
- Server state riêng biệt: **React Query (TanStack Query)** hoặc SWR — rất quan trọng, giúp bạn tận dụng kinh nghiệm API/backend

### 5. Routing
- React Router (nested routes, loaders, protected routes)
- Hoặc Next.js App Router nếu bạn học Next luôn (khuyến khích vì gần với tư duy backend: file-based routing, server components, API routes)

### 6. Networking / Data fetching
- Fetch/Axios, xử lý loading/error state
- Caching, retry, optimistic update (React Query)
- WebSocket ở client (bạn chắc rành ở BE, giờ học phía client dùng thế nào)

### 7. Build tools & tooling
- Vite (khuyến khích học thay vì CRA đã deprecated)
- Bundler cơ bản: hiểu tree-shaking, code splitting, lazy loading
- ESLint, Prettier
- TypeScript trong React (props typing, generic component, hooks typing) — rất nên học vì bạn từ BE quen type rồi

### 8. Testing FE
- Jest / Vitest cho unit test
- React Testing Library (test theo behavior, không test implementation detail)
- Cypress / Playwright cho E2E (bạn có thể liên hệ với kinh nghiệm test API)

### 9. Performance
- Re-render tối ưu: memo, useMemo, useCallback đúng chỗ
- Code splitting, lazy load, Suspense
- Web Vitals: LCP, FID/INP, CLS
- Image optimization

### 10. Fullstack với React
- **Next.js**: SSR, SSG, ISR, Server Components, API Routes — đây là điểm giao thoa mạnh với kinh nghiệm Node của bạn
- Auth trong FE: JWT lưu ở đâu (cookie httpOnly vs localStorage), refresh token flow, session
- Form handling: React Hook Form + Zod/Yup validate

### 11. UI/UX cơ bản (thứ BE dev hay bỏ qua)
- Component library: MUI, Ant Design, shadcn/ui
- Design system, responsive design, accessibility (a11y) cơ bản

### Gợi ý thứ tự học (4-8 tuần tuỳ thời gian)
1. Tuần 1: HTML/CSS ôn lại + JS browser-specific + Flexbox/Grid
2. Tuần 2-3: React core + Hooks + làm 1 CRUD app nhỏ gọi API bạn tự viết ở Node
3. Tuần 4: React Router + React Query + form handling
4. Tuần 5: TypeScript + testing
5. Tuần 6-7: Next.js (SSR/API routes) — build 1 app fullstack hoàn chỉnh
6. Tuần 8: Performance, deploy (Vercel), ôn phỏng vấn

---

## Phần 2: Bộ câu hỏi phỏng vấn Frontend (kèm đáp án ngắn gọn)

### A. HTML/CSS

**1. Box-sizing: content-box vs border-box khác nhau thế nào?**
`content-box` (mặc định): width/height không tính padding & border. `border-box`: width/height đã bao gồm padding & border → dễ tính toán layout hơn, thường set `* { box-sizing: border-box }` global.

**2. Flexbox vs Grid, khi nào dùng cái nào?**
Flexbox: layout 1 chiều (row hoặc column), phù hợp cho component nhỏ (navbar, card list). Grid: layout 2 chiều (row + column cùng lúc), phù hợp cho layout tổng thể trang.

**3. CSS specificity tính như thế nào?**
Thứ tự ưu tiên: inline style > ID > class/attribute/pseudo-class > element/pseudo-element. `!important` override tất cả (nên tránh dùng).

**4. `em` và `rem` khác nhau gì?**
`em` tính theo font-size của phần tử cha gần nhất (có thể compound). `rem` tính theo font-size của root (`html`), ổn định hơn, ít bug hơn khi nest sâu.

**5. Làm sao để responsive design?**
Dùng media queries, đơn vị linh hoạt (%, vw/vh, rem), mobile-first approach, flexbox/grid, `srcset` cho ảnh.

### B. JavaScript (Browser-specific)

**6. Event bubbling và event capturing khác nhau thế nào?**
Capturing: event đi từ root xuống target. Bubbling: event đi từ target lên root (mặc định của hầu hết event listener). `addEventListener(type, fn, true)` để bắt ở phase capturing.

**7. Event delegation là gì, tại sao dùng?**
Gắn 1 listener ở phần tử cha thay vì gắn từng listener cho nhiều con, tận dụng bubbling. Giúp giảm số lượng listener, hoạt động tốt với phần tử được thêm động (dynamic DOM).

**8. Microtask vs Macrotask trong Event Loop trình duyệt?**
Microtask (Promise.then, queueMicrotask) chạy ngay sau mỗi task hiện tại, trước khi render/macrotask tiếp theo. Macrotask (setTimeout, setInterval, I/O) chạy sau khi microtask queue rỗng. Khác Node một chút vì Node có thêm các phase riêng (nextTick, timers, I/O...).

**9. `debounce` và `throttle` khác nhau gì? Khi nào dùng?**
Debounce: chỉ chạy hàm sau khi ngừng gọi 1 khoảng thời gian (dùng cho search input). Throttle: giới hạn chạy hàm tối đa 1 lần mỗi khoảng thời gian, dù gọi liên tục (dùng cho scroll/resize event).

**10. CORS là gì và tại sao trình duyệt chặn?**
Cơ chế bảo mật của trình duyệt chặn request cross-origin trừ khi server đích trả về header cho phép (`Access-Control-Allow-Origin`...). Đây là restriction ở phía **client/browser**, không phải server — bạn từng cấu hình CORS ở Node (Express `cors()` middleware) chính là set header này.

### C. React Fundamentals

**11. Virtual DOM là gì, tại sao React dùng nó?**
Một bản sao nhẹ của DOM thật trong bộ nhớ. React diff (so sánh) virtual DOM cũ và mới để tính toán thay đổi tối thiểu (reconciliation), rồi mới cập nhật DOM thật → tránh thao tác DOM thật tốn kém nhiều lần.

**12. `useEffect` chạy khi nào, dependency array hoạt động ra sao?**
Chạy sau khi component render xong (sau paint). Không có dependency array: chạy mỗi lần render. Mảng rỗng `[]`: chạy 1 lần khi mount. Có dependency: chạy lại khi giá trị trong mảng thay đổi (so sánh bằng Object.is).

**13. Vì sao không nên gọi Hook trong điều kiện (if) hoặc vòng lặp?**
React dựa vào **thứ tự gọi hook** để map state giữa các lần render (linked list nội bộ). Gọi hook có điều kiện làm thứ tự thay đổi giữa các render → state bị lẫn lộn, gây bug khó debug.

**14. `useMemo` và `useCallback` khác nhau gì, khi nào dùng?**
`useMemo` cache **giá trị** tính toán (tránh tính lại tốn kém). `useCallback` cache **function reference** (tránh tạo hàm mới mỗi render, hữu ích khi truyền callback xuống component con được `memo` hoá để tránh re-render thừa).

**15. Controlled vs Uncontrolled component trong form?**
Controlled: giá trị input được React state quản lý (`value` + `onChange`), React là "nguồn sự thật" duy nhất. Uncontrolled: DOM tự quản lý giá trị, lấy giá trị qua `ref` khi cần (ít re-render hơn nhưng khó kiểm soát validate real-time).

**16. `key` prop trong list dùng để làm gì, vì sao không nên dùng index làm key?**
Giúp React nhận diện phần tử nào thay đổi/thêm/xoá giữa các lần render để reconciliation đúng và hiệu quả. Dùng index làm key sẽ sai khi list bị thêm/xoá/sắp xếp lại ở giữa → React map nhầm state/DOM giữa các item.

**17. Re-render trong React xảy ra khi nào?**
Khi: state thay đổi (useState/useReducer), props thay đổi, component cha re-render (con cũng re-render trừ khi được memo), hoặc context value thay đổi (mọi consumer re-render).

**18. `React.memo` hoạt động thế nào, có luôn nên dùng không?**
`memo` skip re-render nếu props không đổi (shallow compare). Không nên lạm dụng — nó tốn chi phí so sánh, chỉ nên dùng cho component render tốn kém hoặc re-render thường xuyên không cần thiết.

**19. Context API có nhược điểm gì so với Redux/Zustand?**
Khi context value thay đổi, **tất cả** component đang consume nó đều re-render (không có selector tối ưu như Redux). Phù hợp cho state ít thay đổi (theme, auth), không phù hợp cho state thay đổi liên tục/phức tạp.

**20. So sánh Redux Toolkit, Zustand, Context API?**
Context: đơn giản, built-in, không cần lib, nhưng re-render toàn bộ và không có dev tools mạnh. Redux Toolkit: predictable, có dev tools, middleware, phù hợp app lớn nhưng boilerplate hơn (dù RTK đã giảm nhiều). Zustand: API tối giản, ít boilerplate, selector tối ưu re-render, không cần Provider bọc toàn app.

### D. Data fetching & Server State

**21. Tại sao nên dùng React Query/SWR thay vì tự fetch bằng useEffect?**
Chúng xử lý sẵn: caching, deduplication request, refetch on focus/reconnect, retry, stale-while-revalidate, loading/error state, pagination/infinite scroll — tránh phải tự viết lại logic dễ bug (race condition, memory leak khi unmount).

**22. Race condition khi fetch trong `useEffect` là gì, xử lý sao?**
Khi user thay đổi input nhanh (VD search), request cũ có thể trả về **sau** request mới → hiển thị data sai. Xử lý bằng cờ `cancelled`/AbortController trong cleanup function của `useEffect`, hoặc dùng React Query (tự xử lý sẵn).

**23. JWT nên lưu ở đâu trong FE: localStorage hay cookie?**
`localStorage`: dễ bị đánh cắp qua XSS (JS đọc được). Cookie `httpOnly` + `Secure` + `SameSite`: an toàn hơn với XSS (JS không đọc được) nhưng cần cẩn thận CSRF → thường kết hợp CSRF token hoặc `SameSite=Strict/Lax`.

**24. Optimistic update là gì?**
Cập nhật UI ngay lập tức (giả định request sẽ thành công) trước khi server phản hồi, để UX mượt hơn. Nếu request fail thì rollback lại state cũ. React Query hỗ trợ sẵn qua `onMutate`/`onError`.

### E. TypeScript trong React

**25. Cách type props cho 1 component?**
```tsx
interface ButtonProps {
  label: string;
  onClick: () => void;
  disabled?: boolean;
}
const Button = ({ label, onClick, disabled }: ButtonProps) => { ... }
```

**26. `interface` vs `type` trong TypeScript, dùng cái nào cho React props?**
Về cơ bản tương tự nhau; `interface` hỗ trợ declaration merging và extend rõ ràng hơn, thường dùng cho object/props. `type` linh hoạt hơn cho union/intersection type. Convention phổ biến: dùng `interface` cho props/state, `type` cho union types.

### F. Performance & Tối ưu

**27. Làm sao để code-splitting trong React?**
Dùng `React.lazy()` + `Suspense` để lazy-load component theo route hoặc theo nhu cầu, giảm bundle size ban đầu (initial load).
```tsx
const Dashboard = React.lazy(() => import('./Dashboard'));
```

**28. Web Vitals quan trọng nhất là gì?**
LCP (Largest Contentful Paint - tốc độ render nội dung chính), INP (Interaction to Next Paint - độ phản hồi tương tác, thay thế FID), CLS (Cumulative Layout Shift - độ ổn định layout).

**29. Làm sao debug re-render không cần thiết trong React?**
Dùng React DevTools Profiler để xem component nào re-render và lý do (highlight updates), kiểm tra props/context có bị tạo mới mỗi render không (object/array/function literal), dùng `memo`/`useMemo`/`useCallback` đúng chỗ.

### G. Testing

**30. React Testing Library khác Enzyme ở điểm nào (triết lý test)?**
RTL khuyến khích test theo **behavior/output** giống người dùng thấy (query theo role, text, label) thay vì test **implementation detail** (state nội bộ, tên method) như Enzyme hay làm → test ít bị vỡ khi refactor code mà UI/behavior không đổi.

### H. Next.js / Fullstack React

**31. SSR, SSG, ISR khác nhau gì?**
SSR (Server-Side Rendering): render HTML mỗi request trên server. SSG (Static Site Generation): render sẵn HTML lúc build, serve static file (nhanh nhất). ISR (Incremental Static Regeneration): SSG nhưng có thể re-generate lại theo thời gian/on-demand mà không cần rebuild toàn bộ site.

**32. React Server Components (RSC) là gì, khác Client Component thế nào?**
RSC render trên server, không gửi JS xuống client (giảm bundle), có thể fetch data trực tiếp/truy cập backend resource, nhưng không dùng được hook tương tác (useState, useEffect, event handler). Client Component (`"use client"`) chạy trên browser, dùng được state/interactivity.

**33. Bạn từng làm Node/Express, Next.js API routes khác gì so với Express route bạn quen?**
Về bản chất tương tự (nhận request, trả response), nhưng Next.js API routes/route handlers tích hợp sẵn trong cùng project với FE (file-based routing, cùng deploy), phù hợp cho BFF (Backend For Frontend) hoặc app nhỏ-vừa; app lớn/phức tạp vẫn nên tách backend riêng như bạn đang quen.

---

## Gợi ý cách trả lời phỏng vấn khi từ BE chuyển sang FE
- Nhấn mạnh lợi thế: hiểu rõ API design, auth flow, performance ở tầng network, dễ làm việc với data fetching/caching (React Query) và Next.js server-side.
- Thành thật về điểm còn yếu (CSS/UI chi tiết, animation...) nhưng cho thấy đã có lộ trình học rõ ràng và đã thực hành qua project cụ thể.
- Chuẩn bị 1-2 project fullstack cá nhân (FE React/Next.js + BE Node bạn tự viết) để demo khi phỏng vấn — đây là điểm cộng rất lớn cho vị trí Fullstack.
