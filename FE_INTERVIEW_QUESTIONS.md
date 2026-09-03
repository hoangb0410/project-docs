# CÂU HỎI PHỎNG VẤN FRONTEND CƠ BẢN

**Ký hiệu**: câu có dấu `*` là bắt buộc phải thuộc, gần như chắc chắn bị hỏi. Ôn 11 câu này trước, phần còn lại đọc lướt để không lạ tai.

Mỗi câu gồm: ý chính cần nói, điểm người phỏng vấn hay hỏi xoáy, và liên hệ với repo nếu có.

---

## Nhóm 1. JavaScript và TypeScript nền tảng

**1. Closure là gì?**

Closure là hàm kết hợp với lexical environment nơi nó được khai báo. Hàm giữ tham chiếu tới các biến của scope cha, nên vẫn đọc và ghi được chúng kể cả khi scope cha đã return.

```js
const makeCounter = () => {
  let count = 0;
  return () => ++count;
};
const counter = makeCounter();
counter(); // 1
counter(); // 2, biến count vẫn sống
```

Trong React, mỗi lần render tạo một bộ hàm mới, mỗi hàm đóng gói giá trị state của đúng lần render đó. Nếu `useEffect` hoặc `setTimeout` giữ callback của render cũ, nó đọc state cũ, gọi là **stale closure**. Đây là lý do dependency array tồn tại.

Hỏi xoáy: closure có gây memory leak không? Có, nếu closure giữ tham chiếu tới object lớn hoặc DOM node mà không bao giờ được giải phóng, ví dụ listener không được remove.

**2. Event loop hoạt động ra sao?**

JS single thread, có một call stack. Code đồng bộ chạy hết trên stack. Khi stack rỗng, event loop lấy task từ hai hàng đợi:

- **Microtask queue**: `Promise.then`, `queueMicrotask`, `MutationObserver`. Được xử lý hết sạch trước khi làm gì khác.
- **Macrotask queue**: `setTimeout`, `setInterval`, I/O, event từ user. Mỗi vòng lặp lấy một task.

Thứ tự: sync → toàn bộ microtask → render (nếu cần) → một macrotask → lặp lại.

```js
setTimeout(() => console.log('timeout'), 0);
Promise.resolve().then(() => console.log('promise'));
console.log('sync');
// sync, promise, timeout
```

Hỏi xoáy: `await` làm gì? Nó tách phần code sau `await` thành một microtask. Vì vậy `async/await` không chặn thread, chỉ tạm dừng hàm đó.

**3. `map` / `forEach` / `reduce` khác gì nhau?**

- `map` trả mảng mới cùng độ dài, không đổi mảng gốc. Dùng khi biến đổi dữ liệu 1:1, và là cách duy nhất render list trong JSX vì cần giá trị trả về.
- `forEach` trả `undefined`, chỉ dùng cho side effect. Không thể `break`, không chain được.
- `reduce` gom mảng về một giá trị bất kỳ: số, object, map. Hay dùng để group by hoặc đếm.
- `filter` lọc, `find` lấy phần tử đầu tiên, `some`/`every` kiểm tra điều kiện.

Hỏi xoáy: vì sao `.map()` không dùng để thực hiện side effect? Vì tạo mảng vô ích và gây hiểu nhầm ý định. Vì sao không dùng `for...in` với mảng? Vì nó duyệt cả property kế thừa và thứ tự không đảm bảo.

**4. `interface` vs `type`?**

Cả hai đều mô tả kiểu và phần lớn trường hợp thay thế được cho nhau, nhưng khác ở khả năng và hành vi:

| | `interface` | `type` |
|---|---|---|
| Object shape | ✅ | ✅ |
| Union, intersection | ❌ | ✅ `A \| B`, `A & B` |
| Primitive, tuple, function alias | ❌ | ✅ `type ID = string`, `type Pair = [number, number]` |
| Mapped / conditional type | ❌ | ✅ `type Keys = keyof T`, `T extends U ? X : Y` |
| Kế thừa | `extends` | dùng `&` |
| Declaration merging | ✅ khai báo lại cùng tên sẽ gộp | ❌ báo lỗi trùng tên |
| Implement bởi class | ✅ | ✅ nếu là object shape |

**Declaration merging** là điểm quan trọng: khai báo `interface Window { myProp: string }` sẽ gộp vào `Window` sẵn có. Cơ chế này dùng để augment type của thư viện thứ ba, ví dụ thêm field vào `DefaultTheme` của styled-components hay `AxiosRequestConfig`. `type` không làm được, khai báo lại là lỗi.

Nhược điểm của merging: dễ vô tình gộp nhầm khi hai file cùng khai báo một tên, nên nhiều codebase quy ước rõ. Quy ước dự án này: object shape dùng `interface` với prefix `I`, `type` chỉ cho union, intersection, utility type.

Hỏi xoáy: `type` có bị giới hạn khi extend không? Intersection `&` với hai field cùng tên nhưng khác kiểu sẽ ra `never` âm thầm, trong khi `interface extends` báo lỗi ngay. Đây là lý do object shape nên dùng `interface`.

**5. Vì sao tránh `any`?**

`any` tắt hoàn toàn type checking cho giá trị đó và lan truyền: gán `any` vào biến khác thì biến đó cũng mất kiểm tra. Bug chỉ lộ ở runtime, mất autocomplete, refactor không an toàn.

Thay thế:
- `unknown` khi thật sự chưa biết kiểu, buộc phải narrow trước khi dùng.
- Generic khi kiểu phụ thuộc đầu vào.
- Khai báo interface thật cho response API. Trong dự án, `any` mới là lỗi review.

Hỏi xoáy: `unknown` khác `any` chỗ nào? `any` cho phép mọi thao tác, `unknown` không cho làm gì cho tới khi kiểm tra kiểu. Cả hai đều nhận mọi giá trị.

**6. Type narrowing là gì?**

Thu hẹp kiểu union về kiểu cụ thể trong một nhánh code, để TypeScript cho phép truy cập property của kiểu đó.

Các cách narrow:
- `typeof x === 'string'` cho primitive.
- `x instanceof Date` cho class.
- `'field' in x` kiểm tra có property.
- **Discriminated union**: các kiểu chung một trường literal, switch theo trường đó.

```ts
interface ISuccess { status: 'success'; data: ICustomer }
interface IFailure { status: 'error'; message: string }
type TResult = ISuccess | IFailure;

if (result.status === 'success') result.data; // TS biết là ISuccess
```

- Type guard tự viết: `const isCustomer = (x: unknown): x is ICustomer => ...`.

Hỏi xoáy: vì sao dùng enum hoặc literal cho trường phân biệt thay vì boolean? Vì mở rộng thêm trạng thái thứ ba không phải viết lại logic.

**7. `Partial`, `Pick`, `Omit`, `Record` dùng khi nào?**

Utility type tạo kiểu mới từ kiểu có sẵn, tránh khai báo lặp:

- `Partial<T>`: mọi field optional. Dùng cho body update hoặc form patch.
- `Required<T>`: ngược lại.
- `Pick<T, 'a' | 'b'>`: chỉ lấy vài field, dùng khi component chỉ cần một phần entity.
- `Omit<T, 'id'>`: bỏ field, dùng cho body create từ entity.
- `Record<K, V>`: object có key thuộc `K`, value kiểu `V`. Ví dụ `Record<EStatus, string>` map enum sang màu, TypeScript ép phải khai báo đủ mọi key.
- `ReturnType<typeof fn>`, `Parameters<typeof fn>`: lấy kiểu từ hàm, hay dùng để type giá trị trả về của custom hook.

Hỏi xoáy: `Pick` và `Omit` cái nào an toàn hơn khi entity thêm field? `Pick` an toàn hơn vì field mới không lọt vào. `Omit` sẽ tự động bao gồm field mới.

**8. `==` vs `===`?**

`==` thực hiện type coercion theo bảng quy tắc phức tạp trước khi so sánh: `'1' == 1` true, `null == undefined` true, `[] == false` true. `===` so sánh cả kiểu và giá trị, không ép kiểu.

Luôn dùng `===`. Trường hợp duy nhất `==` hay được chấp nhận là `x == null` để bắt cả `null` và `undefined`, nhưng `x ?? default` hoặc optional chaining rõ ràng hơn.

Hỏi xoáy: `NaN === NaN` ra gì? `false`. Dùng `Number.isNaN` hoặc `Object.is`.

**9. Shallow copy vs deep copy?**

Shallow copy tạo object mới ở cấp ngoài, nhưng property là object bên trong vẫn trỏ cùng tham chiếu. Spread `{...obj}`, `[...arr]`, `Object.assign` đều là shallow.

```js
const a = { user: { name: 'A' } };
const b = { ...a };
b.user.name = 'B'; // a.user.name cũng thành 'B'
```

Deep copy dùng `structuredClone(obj)` (native, không copy function), hoặc `lodash.cloneDeep`. `JSON.parse(JSON.stringify())` là cách cũ, mất `Date`, `undefined`, `Map`.

Liên hệ React: state phải immutable. Muốn đổi field lồng sâu phải spread từng cấp hoặc dùng Immer. React so sánh tham chiếu `Object.is` để quyết định re-render, nên mutate trực tiếp thì UI không cập nhật.

---

## Nhóm 2. React cơ bản

**\* 10. Component là gì? Props và state khác nhau ra sao?**

Component là hàm nhận `props` và trả về JSX mô tả UI. React gọi hàm này mỗi lần cần render.

| | Props | State |
|---|---|---|
| Nguồn | Cha truyền xuống | Nội bộ component |
| Thay đổi | Read-only trong con | Qua setter, gây re-render |
| Dùng cho | Cấu hình, dữ liệu, callback | Dữ liệu biến đổi theo tương tác |

Nguyên tắc: nếu dữ liệu tính được từ props hoặc state khác thì không lưu thành state, tính trực tiếp trong render. Lưu thừa gây mất đồng bộ.

Trong dự án, component là **presentational**: nhận props, gọi một custom hook, destructure và render. Không có `useState`, `useEffect`, `useQuery` bên trong component.

Hỏi xoáy: có nên copy props vào state không? Không, trừ khi cố ý lấy giá trị khởi tạo một lần. Copy rồi thì props đổi state không đổi theo.

**\* 11. One-way data flow là gì?**

Dữ liệu chảy một chiều từ cha xuống con qua props. Con không sửa props. Muốn thay đổi dữ liệu của cha, con gọi callback mà cha truyền xuống, cha cập nhật state, state mới chảy xuống lại.

Lợi ích: luôn biết dữ liệu đến từ đâu, debug bằng cách đi ngược lên cây. Trái ngược two-way binding của Angular, nơi view và model tự đồng bộ hai chiều và khó trace.

Hệ quả thiết kế: state đặt ở component cha chung thấp nhất của những component cần nó, gọi là **lifting state up**. Nếu quá nhiều cấp thì dùng Context hoặc store.

**\* 12. `useState` hoạt động ra sao?**

`const [value, setValue] = useState(initial)` trả về giá trị hiện tại và setter. Gọi setter đưa component vào hàng đợi re-render, lần render sau `useState` trả giá trị mới.

Điểm cần nói:
- **Setter không đồng bộ**, đọc `value` ngay dòng sau vẫn là giá trị cũ vì render chưa chạy lại.
- **Batching**: nhiều setter trong cùng event handler gộp thành một render.
- **Functional update** `setCount(prev => prev + 1)` khi giá trị mới phụ thuộc giá trị cũ. Gọi `setCount(count + 1)` hai lần chỉ tăng 1 vì cả hai đọc cùng closure.
- **Lazy initial**: `useState(() => expensive())` để hàm khởi tạo chỉ chạy lần đầu.
- Setter với **cùng giá trị** (theo `Object.is`) sẽ bail out, không render.

Hỏi xoáy: vì sao đổi field trong object state mà UI không cập nhật? Vì mutate trực tiếp giữ cùng tham chiếu, React coi là không đổi. Phải tạo object mới.

**\* 13. `useEffect` dùng để làm gì?**

Đồng bộ component với hệ thống bên ngoài React sau khi render đã commit lên DOM: gọi API, đăng ký listener, kết nối socket, đặt timer, thao tác DOM thủ công.

Không dùng `useEffect` để:
- Tính giá trị từ props/state, cứ tính trong render hoặc `useMemo`.
- Xử lý event của user, làm trong handler.
- Đồng bộ state này theo state kia, đó là dấu hiệu state thừa.

Thứ tự: render → commit DOM → chạy cleanup của effect cũ (nếu dependency đổi) → chạy effect mới. `useLayoutEffect` chạy đồng bộ trước khi browser paint, chỉ dùng khi cần đo DOM để tránh nhấp nháy.

Liên hệ BE: effect giống lifecycle hook `onModuleInit`/`onModuleDestroy`, nhưng chạy lại mỗi khi dependency đổi thay vì chỉ một lần.

**\* 14. Dependency array của `useEffect` có ý nghĩa gì?**

Danh sách giá trị mà effect phụ thuộc. React so sánh từng phần tử bằng `Object.is` với lần render trước, khác thì chạy lại effect.

- Không truyền: chạy sau **mỗi** render.
- `[]`: chạy một lần sau mount, cleanup khi unmount.
- `[a, b]`: chạy khi `a` hoặc `b` đổi.

Hai lỗi kinh điển:
- **Thiếu dependency**: effect dùng biến nhưng không khai báo, biến đổi nhưng effect không chạy lại, đọc giá trị cũ. ESLint rule `react-hooks/exhaustive-deps` bắt lỗi này, không nên tắt.
- **Dependency là object/array/function tạo mới mỗi render**: tham chiếu luôn khác nên effect chạy mỗi render. Nếu effect lại set state thì vòng lặp vô hạn. Sửa bằng `useMemo`/`useCallback`, hoặc đưa việc tạo object vào trong effect, hoặc chỉ phụ thuộc primitive bên trong.

Hỏi xoáy: có nên cho `setState` vào dependency? Không cần, setter có tham chiếu ổn định. Tương tự `dispatch` của Redux và `queryClient`.

**\* 15. Cleanup function trong `useEffect` là gì?**

Hàm được return từ effect. React gọi nó ở hai thời điểm: trước khi chạy lại effect do dependency đổi, và khi component unmount.

```ts
useEffect(() => {
  const handler = (data) => setMessages(prev => [...prev, data]);
  socket.on('message', handler);
  return () => socket.off('message', handler);
}, [socket]);
```

Cleanup phải "hoàn tác" đúng thứ effect đã làm: `off` cùng handler đã `on`, `clearTimeout` cùng id, `abort` request, `disconnect` socket, unsubscribe store.

Không cleanup thì mỗi lần dependency đổi lại thêm một listener nữa, handler chạy nhiều lần, hoặc set state trên component đã unmount. Xem `src/hooks/useAppSocket.ts` trong repo để thấy pattern `socket.off` trong cleanup.

Hỏi xoáy: vì sao dev mode effect chạy hai lần? React 18 StrictMode mount, unmount, mount lại để phát hiện effect thiếu cleanup. Effect viết đúng sẽ không có tác dụng phụ khi chạy hai lần.

**16. Vì sao không dùng async function trực tiếp làm callback `useEffect`?**

Async function luôn trả về Promise. `useEffect` kỳ vọng callback trả về `undefined` hoặc một cleanup function. Trả về Promise thì React không có cleanup và cảnh báo.

Cách đúng: khai báo async function bên trong effect rồi gọi, và xử lý trường hợp component unmount trước khi await xong bằng flag `isCancelled` hoặc `AbortController` trong cleanup.

```ts
useEffect(() => {
  const controller = new AbortController();
  const load = async () => {
    const data = await fetchData({ signal: controller.signal });
    setData(data);
  };
  load();
  return () => controller.abort();
}, []);
```

Trong dự án, fetch dữ liệu đi qua TanStack Query nên gần như không cần pattern này, query đã xử lý race condition và unmount.

**17. `useMemo` vs `useCallback`?**

- `useMemo(() => compute(a, b), [a, b])` cache **kết quả** của một phép tính, trả lại giá trị cũ nếu dependency không đổi.
- `useCallback(fn, deps)` cache **tham chiếu hàm**. Tương đương `useMemo(() => fn, deps)`.

Khi nào có ích thật:
- Giá trị được truyền xuống component con bọc `React.memo`. Nếu tham chiếu đổi mỗi render thì `memo` vô nghĩa.
- Giá trị nằm trong dependency array của `useEffect` hoặc hook khác.
- Phép tính thật sự tốn, ví dụ sort/filter list vài nghìn phần tử.
- `value` của Context Provider, để không re-render toàn subtree.

Khi nào vô ích: bọc mọi hàm và mọi object. Bản thân `useMemo`/`useCallback` cũng tốn bộ nhớ và so sánh dependency. Đo trước khi tối ưu.

Hỏi xoáy: `useMemo` có đảm bảo không tính lại không? Không, React có quyền bỏ cache để giải phóng bộ nhớ. Không dùng nó cho tính đúng đắn, chỉ cho hiệu năng.

**\* 18. Vì sao cần `key` khi render list? Vì sao không dùng index?**

Khi reconcile, React so sánh cây cũ và mới. Với danh sách, nó dùng `key` để biết phần tử nào là phần tử nào giữa hai lần render, từ đó reuse DOM node và giữ state nội bộ đúng chỗ.

Dùng index làm key: khi thêm vào đầu, xóa giữa, hoặc sort, index của các phần tử dịch chuyển. React nghĩ phần tử ở vị trí 0 vẫn là phần tử 0 cũ, chỉ cập nhật props. Hậu quả: input đang gõ nhảy sang item khác, animation sai, checkbox tick nhầm, và hiệu năng kém vì phải update toàn bộ thay vì di chuyển node.

Dùng index chấp nhận được khi list tĩnh, không sort/filter/thêm/xóa, và item không có state.

Key phải **ổn định, duy nhất trong siblings**, lấy từ id dữ liệu. Không dùng `Math.random()` vì mỗi render key mới, React unmount và mount lại toàn bộ.

Hỏi xoáy: key có được truyền vào props không? Không, `key` React giữ riêng, muốn dùng trong con phải truyền thêm prop khác.

**19. Controlled vs uncontrolled input?**

- **Controlled**: React state là nguồn sự thật. Input nhận `value` và `onChange`, mỗi ký tự gõ đi qua state. Dễ validate tức thì, format, disable nút theo giá trị, nhưng re-render theo từng phím.
- **Uncontrolled**: DOM giữ giá trị, React đọc khi cần qua `ref` hoặc `FormData`. Ít re-render, nhưng khó phản ứng theo giá trị đang gõ.

React Hook Form kết hợp cả hai: field HTML native dùng `register` (uncontrolled, hiệu năng cao), field từ thư viện UI như Ant Design dùng `Controller` (controlled). Dự án quy ước mọi field đi qua `Controller` để thống nhất error rendering.

Hỏi xoáy: warning "changing an uncontrolled input to controlled" là gì? Khi `value` ban đầu là `undefined` rồi có giá trị. Khởi tạo `''` thay vì `undefined`.

**\* 20. Component re-render khi nào?**

Component chạy lại hàm render khi:
1. **State của nó** đổi (theo `Object.is`).
2. **Component cha** re-render. Mặc định mọi con đều render lại, **bất kể props có đổi hay không**.
3. **Context** nó subscribe qua `useContext` đổi value.
4. Hook nó dùng trigger, ví dụ `useQuery` nhận data mới, `useSelector` chọn ra giá trị khác.

Props đổi không tự gây re-render, mà là cha re-render kéo theo. `React.memo` chặn điều này bằng cách so sánh shallow props, chỉ có ích khi props ổn định về tham chiếu.

Re-render không đồng nghĩa đụng DOM. React diff virtual DOM và chỉ commit phần thay đổi. Re-render thừa tốn CPU, không nhất thiết tốn DOM.

Cách giảm re-render thừa:
- Đặt state gần nơi dùng nhất, tách component nhỏ.
- Truyền JSX qua `children` để phần đó không bị render lại theo cha.
- `React.memo` + `useCallback`/`useMemo` cho props.
- Tách Context theo tần suất đổi, không đặt giá trị đổi liên tục vào Context.

Hỏi xoáy: vì sao đặt `activeId` của drag vào Context là sai? Vì mỗi pixel kéo là một lần đổi value, toàn subtree re-render. Dự án để những giá trị đó làm local prop.

**21. Prop drilling là gì và cách tránh?**

Truyền một prop qua nhiều tầng component chỉ để đến được đích, trong khi các tầng giữa không dùng nó. Hậu quả: signature phình to, refactor khó, thêm field là phải sửa cả chuỗi.

Cách tránh:
- **Component composition**: cha render sẵn con rồi truyền qua `children` hoặc slot prop, thay vì truyền dữ liệu để con tự render.
- **Context** module-scoped: Provider ở đầu module, `use[Feature]Context()` ở leaf, throw nếu dùng ngoài Provider.
- **Store** (Redux, Zustand) cho state toàn app.
- **TanStack Query**: leaf tự gọi `useQuery` với cùng key, đọc từ cache, không cần cha truyền data.

Quy ước dự án: tối đa 2 tầng. Sâu hơn thì dùng Context. Xem `src/modules/reservations/context/ReservationsKanbanContext.tsx`.

**\* 22. Custom hook là gì?**

Hàm JavaScript tên bắt đầu bằng `use`, bên trong gọi các hook khác. Nó là cách **tái sử dụng logic có state**, không phải tái sử dụng state. Hai component gọi cùng custom hook có hai bản state độc lập.

Trong dự án, mỗi component đi kèm một `use[X]Hooks.ts` chứa toàn bộ business logic: state, react-query, react-hook-form, handlers, effects. Hook return một object, component destructure và render.

```ts
export const useAddCustomerHooks = (props) => {
  const { t } = useTranslation();
  const { showServerSuccessMsg, showServerErrorMsg } = useAppToast();
  const { control, handleSubmit, reset } = useForm({ resolver: yupResolver(schema) });
  const { mutate: mutateAddCustomer, isPending } = useAddCustomerMutation({
    configs: { onSuccess: (res) => { showServerSuccessMsg(res); reset(); }, onError: showServerErrorMsg },
  });
  const handleAddCustomer = handleSubmit((values) => mutateAddCustomer({ body: values }));
  return { t, control, isPending, handleAddCustomer };
};
```

Lợi ích: component thuần trình bày, dễ đọc; logic test bằng `renderHook` không cần render UI; đổi UI không đụng logic.

Hỏi xoáy: custom hook khác helper function ở đâu? Helper là hàm thuần, không gọi hook, gọi được ở bất kỳ đâu. Custom hook phải tuân quy tắc hooks và chỉ gọi trong component hoặc hook khác.

**23. Quy tắc của hooks?**

1. Chỉ gọi hook ở **top level** của function component hoặc custom hook. Không trong `if`, vòng lặp, hàm lồng, sau `return` sớm.
2. Chỉ gọi từ **React function** (component hoặc custom hook), không từ hàm thường hoặc class.

Lý do: React không lưu hook theo tên mà theo **thứ tự gọi** trong một linked list. Render này gọi 3 hook, render sau gọi 2 hook do điều kiện, thì hook thứ 2 nhận state của hook thứ 3. Thứ tự phải cố định qua mọi render.

Muốn hook chạy có điều kiện: đặt điều kiện bên trong hook (`enabled: !!id` của `useQuery`), hoặc tách component con render có điều kiện.

**24. `useRef` dùng để làm gì?**

Trả về object `{ current }` tồn tại xuyên suốt đời component. Đổi `.current` **không gây re-render** và không cần dependency.

Ba mục đích:
- **Tham chiếu DOM**: `<input ref={inputRef} />` rồi `inputRef.current.focus()`.
- **Giữ giá trị mutable không liên quan render**: socket instance, timer id, `AbortController`, giá trị trước đó, flag `isMounted`.
- **Tránh stale closure**: lưu callback mới nhất vào ref để listener đăng ký một lần vẫn gọi được logic mới nhất.

So với `useState`: state đổi thì render, ref đổi thì không. Nếu giá trị hiển thị lên UI thì dùng state, nếu chỉ dùng trong logic thì ref.

Hỏi xoáy: đọc `ref.current` trong render có an toàn không? Không nên, vì ref có thể chưa gắn hoặc đổi mà UI không cập nhật. Đọc trong effect hoặc handler.

**25. Context là gì và khi nào dùng?**

Cơ chế truyền giá trị cho cả subtree mà không qua props. `createContext` → `<Provider value>` bao subtree → `useContext` ở bất kỳ con nào.

Khi Provider `value` đổi (theo `Object.is`), **mọi** component đang `useContext` đều re-render, không có selective subscribe như Redux.

Dùng cho: theme, locale, user hiện tại, config của một module, handler dùng chung. Không dùng cho state đổi liên tục hoặc state nhiều người ghi phức tạp.

Best practice trong dự án:
- Memo `value` bằng `useMemo`, không tạo object inline trong JSX.
- Custom hook `use[Feature]Context()` throw khi dùng ngoài Provider, báo lỗi sớm.
- Tách Context theo tần suất đổi: context "config" ổn định và context "state" hay đổi.

Hỏi xoáy: Context có thay Redux được không? Cho state nhỏ, ít đổi thì được. Redux có selector, middleware, devtools, và chỉ re-render component chọn ra giá trị đổi.

**26. Error Boundary là gì?**

Component bắt lỗi JavaScript xảy ra trong **quá trình render**, lifecycle và constructor của cây con, hiển thị fallback UI thay vì unmount toàn app. Hiện chỉ viết được bằng class component với `getDerivedStateFromError` và `componentDidCatch`, hoặc dùng `react-error-boundary`.

Không bắt: lỗi trong event handler, code async (`setTimeout`, promise), SSR, lỗi của chính boundary. Những chỗ đó dùng `try/catch` hoặc `onError` của query/mutation.

Đặt boundary theo tầng: một cái quanh app, và một cái quanh mỗi vùng độc lập (widget, page) để lỗi một vùng không kéo cả trang. Dự án có `AppErrorBoundary` + `AppFallBack`.

**27. Lazy loading component ra sao?**

`const Page = React.lazy(() => import('./Page'))` khiến bundler tách file thành chunk riêng, chỉ tải khi component được render lần đầu. Bọc bằng `<Suspense fallback={<Spinner />}>` để hiển thị gì trong lúc tải.

Thường tách theo route: user vào trang customers không phải tải code của campaigns. Giảm bundle ban đầu, cải thiện thời gian tương tác đầu tiên.

Lưu ý: `lazy` chỉ nhận default export; named export phải `.then(m => ({ default: m.Page }))`. Kết hợp Error Boundary để xử lý chunk tải lỗi sau khi deploy bản mới.

---

## Nhóm 3. Quản lý data và state

**28. Server state vs client state?**

- **Server state**: dữ liệu sở hữu bởi server, FE chỉ giữ bản sao có thể cũ. Cần cache, biết khi nào stale, refetch, dedupe, đồng bộ nhiều nơi. TanStack Query sinh ra cho việc này.
- **Client state**: dữ liệu chỉ tồn tại trên UI, không có ở server: modal đang mở, tab đang chọn, filter chưa submit, form đang gõ. Dùng `useState`, Context, Redux.

Sai lầm phổ biến: nhét response API vào Redux rồi tự viết loading/error/refetch. Đó là tái phát minh TanStack Query với nhiều bug hơn.

Liên hệ BE: server state giống read-through cache trước DB; client state giống session của một request.

**29. TanStack Query giải quyết vấn đề gì?**

Không dùng nó, mỗi màn phải tự viết: `useState` cho data, loading, error; `useEffect` gọi API; xử lý unmount, race condition khi params đổi nhanh; refetch khi quay lại tab; hai component cùng cần dữ liệu thì gọi hai lần.

TanStack Query cung cấp:
- **Cache theo key**, hai component cùng key dùng chung một request và một bản dữ liệu.
- **Dedupe** request song song.
- Trạng thái `isLoading`, `isError`, `isFetching`, `data` chuẩn.
- **Stale-while-revalidate**: trả cache ngay, refetch nền.
- Refetch khi focus window, reconnect, interval.
- **Invalidate** theo key sau mutation.
- Retry, pagination, infinite query, optimistic update.

**\* 30. `useQuery` vs `useMutation`?**

| | `useQuery` | `useMutation` |
|---|---|---|
| Mục đích | Đọc (GET) | Ghi (POST/PUT/DELETE) |
| Kích hoạt | Tự động khi mount và khi key đổi | Thủ công qua `mutate()` / `mutateAsync()` |
| Cache | Có, theo `queryKey` | Không |
| Trả về | `data`, `isLoading`, `isFetching`, `error`, `refetch` | `mutate`, `isPending`, `error`, `data` |
| Callback | Không nên dùng `onSuccess` trong hook | `onSuccess`, `onError`, `onSettled`, `onMutate` |

`mutate` fire-and-forget, xử lý kết quả trong callback. `mutateAsync` trả Promise, dùng khi cần `await` rồi làm tiếp trong cùng luồng.

Quy ước dự án: mutation hook trong `src/react-query/` không chứa side effect, chỉ nhận `configs` và spread. Module hook truyền `onSuccess`/`onError` qua `configs`, trong đó toast, `reset()`, đóng modal, invalidate.

Hỏi xoáy: `isLoading` khác `isFetching`? `isLoading` là lần tải đầu chưa có data. `isFetching` là bất kỳ lần fetch nào, kể cả refetch nền khi đã có data.

**\* 31. Query key dùng để làm gì?**

Là **định danh cache** của một query, dạng mảng: `['customers', { page, search }]`. Giống cache key trong Redis.

Vai trò:
- Hai `useQuery` cùng key chia sẻ cùng cache entry và cùng request.
- **Key đổi thì fetch mới**: params nằm trong key nên đổi page/filter tự động tải lại, không cần `useEffect`.
- **Invalidate theo prefix**: `invalidateQueries({ queryKey: ['customers'] })` đánh dấu stale mọi key bắt đầu bằng `'customers'`, kể cả `['customers', {page: 2}]`.
- Optimistic update đọc/ghi cache qua `getQueryData(key)` / `setQueryData(key)`.

Nguyên tắc: mọi biến mà `queryFn` dùng phải nằm trong key, nếu không cache trả dữ liệu sai cho params khác. Key phải serializable và thứ tự ổn định. Dự án dùng hằng số `QK_GET_XXX` để tránh gõ sai.

Hỏi xoáy: vì sao key là mảng không phải string? Để match theo prefix và so sánh sâu object params, `{page: 1, size: 10}` và `{size: 10, page: 1}` là cùng key.

**32. Invalidate cache sau mutation làm sao?**

Trong `onSuccess` của mutation, gọi `queryClient.invalidateQueries({ queryKey: [QK_GET_CUSTOMERS] })`.

Điều xảy ra: mọi query match prefix bị đánh dấu **stale**. Query nào đang có component mount sẽ **refetch ngay**, query không active sẽ refetch khi được dùng lại. Trong lúc refetch, UI vẫn hiển thị data cũ, không nhấp nháy loading.

Chọn phạm vi: invalidate rộng (`['customers']`) đơn giản và an toàn, invalidate hẹp (`['customers', id]`) tiết kiệm request. Thường invalidate cả list và detail của entity vừa sửa.

Thay thế khi biết chính xác kết quả: `setQueryData` ghi thẳng response vào cache, không cần refetch.

Trong dự án, invalidate nằm ở module hook, không nằm trong `src/react-query/`.

**33. Optimistic update là gì?**

Cập nhật cache ngay khi user thao tác, coi như server sẽ thành công, rồi sửa lại nếu thất bại. UI phản hồi 0ms thay vì chờ round-trip.

Bốn bước với `useMutation`:
1. `onMutate`: `cancelQueries` để refetch đang chạy không ghi đè; `getQueryData` snapshot cache cũ; `setQueryData` ghi giá trị mong đợi; return snapshot làm context.
2. `onError(err, vars, context)`: `setQueryData(key, context.snapshot)` rollback.
3. `onSuccess`: có thể `setQueryData` bằng response thật.
4. `onSettled`: `invalidateQueries` để đồng bộ lại với server dù thành công hay lỗi.

Dùng khi thao tác nhanh, tỉ lệ thành công cao, lỗi dễ hoàn tác: toggle, kéo thả kanban, đánh dấu đã đọc. Không dùng cho thanh toán hoặc tạo entity cần id từ server.

**34. `staleTime` vs `gcTime`?**

- `staleTime` (mặc định 0): thời gian data được coi là **fresh**. Trong khoảng này, component mount mới hoặc window focus **không** refetch, dùng cache luôn. Hết thời gian thì data **stale**, vẫn hiển thị nhưng sẽ refetch nền khi có trigger.
- `gcTime` (mặc định 5 phút, tên cũ `cacheTime`): sau khi **không còn component nào** dùng query, cache giữ thêm bao lâu trước khi xóa. Quay lại trong khoảng này thấy data ngay rồi refetch nền.

Chỉnh `staleTime` để giảm request cho dữ liệu ít đổi (danh sách venue, config). `gcTime` để cân bằng bộ nhớ và trải nghiệm quay lại màn cũ.

Hỏi xoáy: `staleTime: Infinity` nghĩa là gì? Không bao giờ tự refetch, chỉ refetch khi invalidate hoặc gọi `refetch()` thủ công.

**35. Khi nào dùng Redux?**

Khi client state cần **chia sẻ toàn app**, được đọc và ghi từ nhiều nơi không liên quan trong cây, và có logic cập nhật phức tạp: user đăng nhập, venue đang chọn, quyền, cài đặt UI toàn cục, trạng thái socket.

Không dùng Redux cho: server state (đã có TanStack Query), state của một màn (local state), state của một module (Context), form (React Hook Form).

Redux so với Context: có `useSelector` nên component chỉ re-render khi phần nó chọn đổi; có middleware, devtools time-travel, và tách logic ra khỏi component. Chi phí là boilerplate và học thêm khái niệm.

**36. Redux Toolkit khác Redux thuần thế nào?**

Redux thuần phải tự viết action type constants, action creator, reducer với switch case và spread immutable, cấu hình store với middleware và devtools. Rất nhiều boilerplate.

Redux Toolkit (RTK) là cách viết chính thức được khuyến nghị:
- `createSlice` gộp state, reducer, action creator vào một chỗ, tự sinh action type.
- Dùng **Immer** bên dưới: viết `state.items.push(x)` trong reducer, Immer tạo bản immutable mới. Không spread thủ công.
- `configureStore` bật sẵn devtools, thunk, và middleware kiểm tra mutate/serializable trong dev.
- `createAsyncThunk` cho async, `RTK Query` cho data fetching (dự án dùng TanStack Query thay).

Hỏi xoáy: reducer phải pure, vì sao Immer cho phép mutate? Immer ghi lại thao tác trên proxy rồi sinh object mới, reducer thật vẫn pure.

**37. Vì sao dự án tách 3 lớp api / react-query / hooks?**

```
src/api/[feature]/          axios call thuần, nhận params trả Promise<response>
src/react-query/[feature]/  bọc api thành useQuery/useMutation, không side effect
src/modules/[feature]/hooks/ business logic: toast, invalidate, form, modal
```

Lý do:
- **Tách trách nhiệm**: lớp api không biết React; lớp react-query không biết UI phản hồi thế nào; module hook quyết định UX.
- **Tái sử dụng**: một `useGetCustomersQuery` dùng ở nhiều module với cách xử lý success khác nhau. Nếu toast nằm trong react-query thì mọi nơi buộc phải toast giống nhau.
- **Test**: mock lớp api để test react-query, mock react-query để test module hook.
- **Thay đổi dễ**: đổi endpoint chỉ sửa `api`, đổi cách cache chỉ sửa `react-query`.

Tương tự NestJS: Repository → Service → Controller. Không cho controller gọi thẳng repository.

---

## Nhóm 4. Form và validation

**38. React Hook Form giải quyết vấn đề gì?**

Form thuần React với `useState` cho mỗi field: mỗi phím gõ re-render toàn form, validation tự viết, trạng thái touched/dirty/submitting tự quản, reset thủ công.

React Hook Form (RHF):
- Dùng **ref và uncontrolled** với input native, gõ không re-render form. Chỉ component subscribe (`watch`, `formState`) mới render.
- Quản lý sẵn `errors`, `isDirty`, `touchedFields`, `isSubmitting`, `isValid`.
- Tích hợp schema validation qua `resolver` (Yup, Zod).
- `reset`, `setValue`, `getValues`, `trigger` để điều khiển form từ ngoài.
- Nested field và mảng field qua `useFieldArray`.

**39. `Controller` dùng khi nào?**

`register` chỉ gắn được vào input native qua `ref`. Component của thư viện UI (Ant Design `Select`, `DatePicker`, `Switch`) là controlled, nhận `value`/`onChange` và không expose ref theo cách RHF cần.

`Controller` là cầu nối: nhận `name` và `control`, render prop `field` chứa `value`, `onChange`, `onBlur`, `ref`, và `fieldState` chứa `error`. Ta rải `field` vào component UI.

```tsx
<Controller
  name="status"
  control={control}
  render={({ field, fieldState }) => (
    <AppSingleSelect {...field} options={options} errors={fieldState.error?.message} />
  )}
/>
```

Quy ước dự án: mọi field đều đi qua `Controller`, kể cả input text, để thống nhất cách render label/error trong `App*` component.

**40. `control`, `errors`, `reset`, `handleSubmit` là gì?**

Từ `useForm()`:
- `control`: object nội bộ giữ registry field và state, truyền cho `Controller` và `useFieldArray`. Không đọc trực tiếp.
- `formState.errors`: object lỗi theo tên field, `errors.email?.message`. Chỉ cập nhật khi validate chạy (theo `mode`: `onSubmit`, `onBlur`, `onChange`).
- `handleSubmit(onValid, onInvalid?)`: trả về handler cho `onSubmit`. Nó `preventDefault`, chạy validation toàn form, chỉ gọi `onValid(values)` khi hợp lệ.
- `reset(values?)`: đưa form về `defaultValues` hoặc giá trị truyền vào, xóa errors và dirty. Gọi sau khi mutation thành công và khi đóng modal.
- `watch('field')`: subscribe giá trị để render có điều kiện. Dùng tiết kiệm vì gây re-render.
- `setValue`, `getValues`, `trigger`: ghi, đọc, validate thủ công.

Hỏi xoáy: `reset()` sau khi fetch data để edit làm sao? Truyền `values` vào `useForm` hoặc gọi `reset(data)` trong `useEffect` khi data về. Không set `defaultValues` bằng data chưa có.

**41. Yup dùng làm gì?**

Thư viện khai báo **schema validation** dạng chain: `yup.object({ email: yup.string().email().required() })`. Schema mô tả dữ liệu hợp lệ trông thế nào, thay vì viết `if` từng field.

Nối với RHF qua `resolver: yupResolver(schema)`. RHF gọi schema khi validate, map lỗi vào `errors` đúng tên field.

Điểm nên nói:
- Schema đặt riêng trong `src/validations/`, tái sử dụng và test độc lập.
- Message lỗi đi qua `t()` để i18n.
- `.when()` cho validation phụ thuộc field khác, `.transform()` để trim hoặc ép kiểu.
- `yup.InferType<typeof schema>` sinh TypeScript type từ schema, không khai báo lặp.

Tương tự `class-validator` + DTO trong NestJS, nhưng chạy phía client.

**42. Vì sao validate ở FE dù BE đã validate?**

Hai mục đích khác nhau:
- **FE validate cho trải nghiệm**: phản hồi ngay khi gõ hoặc blur, không tốn một round-trip để biết email sai định dạng, giảm request lỗi tới server, hướng dẫn user sửa từng field.
- **BE validate cho tính đúng và bảo mật**: FE có thể bị bypass bằng curl hoặc devtools, client cũ có thể gửi dữ liệu lệch. BE là hàng rào cuối.

Không bao giờ bỏ validate BE vì đã có FE. Ngược lại có thể bỏ validate FE nhưng UX kém.

Hỏi xoáy: làm sao đồng bộ rule hai bên? Chia sẻ schema qua package chung, hoặc BE trả lỗi theo field để FE map vào `setError`.

**43. Vì sao trim input ở `onBlur`?**

Khoảng trắng đầu/cuối làm sai dữ liệu: `"abc@x.com "` fail email validation, hai tên trùng nhưng khác khoảng trắng, search không match.

Trim ở `onChange` thì user không gõ được dấu cách giữa hai từ vì mỗi ký tự cách vừa gõ bị xóa ngay. Trim ở `onBlur` (rời field) để user gõ tự nhiên, dữ liệu sạch khi rời đi.

Cách làm với `Controller`: trong `onBlur` gọi `field.onChange(field.value.trim())` rồi `field.onBlur()`. Hoặc `transform` trong Yup schema, nhưng khi đó giá trị hiển thị vẫn có khoảng trắng.

---

## Nhóm 5. Styling và UI

**44. Box model là gì?**

Mỗi element là một hộp gồm 4 lớp từ trong ra: **content** → **padding** → **border** → **margin**. Margin không tính vào kích thước hộp và có thể collapse với margin của element kề.

`box-sizing`:
- `content-box` (mặc định): `width` chỉ là content. Đặt `width: 100px; padding: 10px; border: 1px` thì hộp thực rộng 122px.
- `border-box`: `width` bao gồm padding và border. Hộp đúng 100px. Gần như mọi reset CSS đặt `* { box-sizing: border-box }`.

Hỏi xoáy: margin collapse là gì? Hai margin dọc kề nhau gộp thành một, lấy giá trị lớn hơn. Không xảy ra với flex/grid item.

**45. Flexbox vs Grid?**

- **Flexbox**: xếp item theo **một trục** (hàng hoặc cột), item co giãn theo nội dung. Dùng cho toolbar, nav, căn giữa, card header với title trái và action phải. Thuật ngữ: `justify-content` theo trục chính, `align-items` theo trục phụ, `gap`, `flex: 1`.
- **Grid**: định nghĩa **hàng và cột** trước, đặt item vào ô. Dùng cho layout trang (sidebar + content), lưới card đều nhau, form nhiều cột. Thuật ngữ: `grid-template-columns: repeat(3, 1fr)`, `grid-area`, `auto-fill`/`auto-fit` cho responsive không cần media query.

Chọn: bố cục theo nội dung và một chiều → flex. Bố cục theo khung định sẵn và hai chiều → grid. Kết hợp cả hai là bình thường: grid cho trang, flex trong từng ô.

**46. CSS specificity là gì?**

Khi nhiều rule cùng áp lên một element, browser chọn rule có độ ưu tiên cao hơn. Tính bằng bộ (a, b, c):
- a: số id selector `#header`
- b: số class, attribute, pseudo-class `.btn`, `[type]`, `:hover`
- c: số tag, pseudo-element `div`, `::before`

So sánh từ trái sang phải. `#x` thắng `.a.b.c.d`. Bằng nhau thì rule khai báo sau thắng. Inline `style` thắng mọi selector. `!important` thắng tất cả kể cả inline.

Vì sao tránh `!important`: phá vỡ thứ tự tự nhiên, muốn override phải dùng `!important` khác với specificity cao hơn, leo thang không kiểm soát. Thay bằng selector cụ thể hơn hoặc sắp xếp thứ tự CSS.

Liên hệ dự án: `AppText` có class riêng, muốn override từ cha thì dùng descendant selector `.wrapper .text-class {}` trong `Styled*`, đủ specificity mà không cần `!important`.

**47. styled-components hoạt động ra sao?**

Viết CSS trong template literal, thư viện hash nội dung thành class name unique (`.sc-abc123`), inject vào `<style>` trong head, và trả về component gắn class đó.

```ts
const StyledCard = styled.div<{ $active: boolean }>`
  padding: 16px;
  border-color: ${({ $active }) => ($active ? Colors.blue_20 : Colors.gray_90)};
`;
```

Điểm nên nói:
- **Scope tự động**, không đụng class của component khác.
- **Dynamic style theo props**: mỗi bộ giá trị props sinh một class.
- Prefix **`$` (transient props)**: styled-components không truyền prop bắt đầu bằng `$` xuống DOM. Không có `$` thì `active` lọt vào `<div active>` gây warning và HTML sai.
- Nested selector và `&` cho pseudo-class, class con.
- Theme qua `ThemeProvider`.

Nhược điểm: sinh CSS ở runtime, tốn CPU hơn CSS tĩnh; mỗi giá trị props mới là một class mới nên tránh dùng cho giá trị đổi liên tục như tọa độ drag.

Quy ước dự án: một styled root mỗi file, sub-element style qua nested class, không đẻ nhiều styled component nhỏ.

**48. Vì sao wrap Ant Design thành `App*` component?**

- **Một điểm cấu hình**: style, kích cỡ, màu theo design system đặt một lần. Đổi theme sửa một file.
- **Ép quy ước**: `AppInput` luôn nhận `label`, `required`, `errors` và render lỗi thống nhất. Không ai quên hiển thị lỗi.
- **Che API thư viện**: đổi từ Ant Design sang thư viện khác hoặc nâng major version chỉ sửa lớp `App*`.
- **Thêm hành vi chung**: loading state, accessibility (`aria-label`), tooltip cho icon button.
- **Giới hạn bề mặt**: chỉ expose props team cần, tránh mỗi nơi dùng một kiểu.

Đánh đổi: thêm một lớp, prop mới phải thêm vào wrapper. Phù hợp với app lớn nhiều người làm.

**49. Responsive làm sao?**

- **Media query** `@media (max-width: 768px)` đổi layout theo breakpoint. Mobile-first: viết style mobile mặc định, `min-width` cho màn lớn.
- **Đơn vị tương đối**: `%`, `rem`, `vw`/`vh`, `clamp()` cho font-size co giãn.
- **Layout tự co**: flex `wrap`, grid `repeat(auto-fit, minmax(240px, 1fr))` tự xuống dòng không cần media query.
- `max-width: 100%` cho ảnh và bảng; bảng rộng bọc trong container `overflow-x: auto`.
- `<meta name="viewport" content="width=device-width">` bắt buộc.
- Container query `@container` cho component responsive theo cha thay vì theo viewport.

Hỏi xoáy: adaptive khác responsive? Adaptive tải layout khác hẳn cho từng thiết bị. Responsive một layout co giãn liên tục.

**50. `rem` vs `px`?**

- `px` cố định tuyệt đối. Dễ khớp design, nhưng không scale khi user tăng cỡ chữ trong browser.
- `rem` bằng font-size của `<html>`, mặc định 16px. User đổi cỡ chữ thì mọi thứ `rem` scale theo, tốt cho accessibility. Đặt `html { font-size: 62.5% }` để 1rem = 10px cho dễ tính.
- `em` theo font-size của element hiện tại, lồng nhau thì nhân lên, khó kiểm soát. Chỉ dùng cho padding tỉ lệ theo chữ của chính element.

Dự án dùng `px` qua props typography của `AppText`. Chấp nhận được với web app nội bộ dùng trên desktop, nhưng hãy nói được đánh đổi.

---

## Nhóm 6. Web và bảo mật

**51. Browser render trang như thế nào?**

1. Parse HTML → **DOM tree**. Gặp `<script>` không `defer`/`async` thì dừng parse để chạy.
2. Parse CSS → **CSSOM**. CSS chặn render (render-blocking) vì cần biết style trước khi vẽ.
3. Gộp DOM + CSSOM → **render tree**, bỏ node `display: none`.
4. **Layout / reflow**: tính vị trí, kích thước từng box.
5. **Paint**: vẽ pixel theo lớp.
6. **Composite**: GPU ghép các lớp.

Hệ quả hiệu năng:
- Đổi `width`, `margin`, thêm node → reflow + paint, tốn.
- Đổi `color`, `background` → chỉ paint.
- Đổi `transform`, `opacity` → chỉ composite, rẻ nhất. Animation nên dùng hai property này.
- Đọc `offsetHeight` rồi ghi style trong vòng lặp gây **layout thrashing**.

React batch update DOM trong commit phase để giảm reflow, nhưng không cứu được CSS animation viết sai.

**52. CORS là gì từ góc nhìn FE?**

Same-origin policy: JS chỉ đọc được response từ cùng origin (scheme + host + port). CORS là cơ chế server **cho phép** origin khác đọc bằng header `Access-Control-Allow-Origin`.

Điểm FE cần hiểu:
- **Browser chặn, không phải server**. Request vẫn tới server, chỉ là JS không đọc được response. Postman không có CORS.
- **Preflight** `OPTIONS` tự động gửi trước request "không đơn giản": method ngoài GET/POST/HEAD, header tùy chỉnh như `Authorization`, `Content-Type: application/json`. Server phải trả `Allow-Methods`, `Allow-Headers`.
- Cookie cross-origin cần `credentials: 'include'` ở FE và `Access-Control-Allow-Credentials: true` + origin cụ thể (không được `*`) ở BE.
- FE không sửa được CORS. Khi dev dùng **proxy** của Vite (`server.proxy`) để request cùng origin.

Lợi thế của bạn: đã cấu hình `enableCors` trong NestJS, nói rõ header nào là của ai.

**53. Cookie vs localStorage vs sessionStorage?**

| | Cookie | localStorage | sessionStorage |
|---|---|---|---|
| Dung lượng | ~4KB | ~5–10MB | ~5MB |
| Gửi kèm request | Tự động mọi request cùng domain | Không | Không |
| JS đọc được | Có, trừ `HttpOnly` | Có | Có |
| Tồn tại | Theo `Expires`/`Max-Age` | Vĩnh viễn | Đến khi đóng tab |
| Chia sẻ tab | Có | Có | Không |

Lưu token:
- `localStorage`: tiện, nhưng bất kỳ XSS nào đọc được và gửi token đi. Không chống được.
- Cookie `HttpOnly; Secure; SameSite=Lax/Strict`: JS không đọc được nên XSS không lấy được token. Đổi lại phải phòng CSRF, và không tự gắn `Authorization` header mà dựa vào cookie tự gửi.

Khuyến nghị chung: access token ngắn hạn trong memory, refresh token trong cookie `HttpOnly`. Với app nội bộ nhiều team vẫn dùng localStorage, biết đánh đổi là đủ.

**54. XSS là gì và cách phòng?**

Cross-Site Scripting: kẻ tấn công đưa được script vào trang mà user khác xem, script chạy với quyền của user đó: đọc cookie/localStorage, gọi API thay user, đổi nội dung trang.

Ba loại: **stored** (lưu trong DB, ví dụ comment chứa `<script>`), **reflected** (từ URL param render ra), **DOM-based** (JS client tự chèn HTML từ input).

React phòng mặc định: `{userInput}` trong JSX được **escape**, `<script>` hiển thị thành text. Lỗ hổng chỉ mở khi:
- `dangerouslySetInnerHTML` với HTML chưa sanitize. Luôn `DOMPurify.sanitize(html)` trước, đây là quy tắc cứng của dự án cho nội dung từ `AppQuill` hoặc email template.
- `href={userUrl}` với `javascript:` URL.
- `eval`, `new Function`, chèn script tag động.

Thêm lớp bảo vệ: CSP header, cookie `HttpOnly`, escape phía server.

**55. CSRF là gì?**

Cross-Site Request Forgery: user đang đăng nhập site A (cookie còn sống), vào site độc B. B tạo form hoặc `<img src>` gửi request tới A. Browser tự đính cookie của A, server A tưởng user chủ động gọi.

Chỉ nguy hiểm khi xác thực dựa trên **cookie tự gửi**. Bearer token trong `Authorization` header do JS gắn thì site B không gắn được, miễn nhiễm CSRF (nhưng lại lộ với XSS, xem câu 53).

Phòng:
- Cookie `SameSite=Lax` (mặc định Chrome) hoặc `Strict`: không gửi cookie khi request từ site khác.
- **CSRF token**: server phát token vào form/header, request phải mang đúng token. Site B không đọc được token do same-origin.
- Kiểm tra `Origin`/`Referer` header.
- Không dùng GET cho hành động thay đổi dữ liệu.

**56. Debounce vs throttle?**

Cả hai giới hạn số lần gọi hàm khi event bắn liên tục.

- **Debounce**: đợi user **ngừng** thao tác một khoảng `wait` rồi chạy **một lần**. Mỗi event mới reset đồng hồ. Dùng cho search input (gọi API sau khi ngừng gõ 300ms), auto-save, validate khi gõ, resize window để tính layout cuối.
- **Throttle**: chạy **tối đa một lần** mỗi khoảng `wait`, dù event bắn bao nhiêu. Dùng cho scroll (infinite scroll, sticky header), mousemove, drag, tracking analytics.

Nhớ nhanh: debounce quan tâm **kết quả cuối**, throttle quan tâm **cập nhật đều đặn trong quá trình**.

Trong React: dùng `lodash.debounce` bọc trong `useMemo`/`useRef` để không tạo lại mỗi render (tạo lại thì timer reset, debounce vô hiệu), và `cancel()` trong cleanup.

**57. Virtualization là gì?**

Chỉ render các item **nằm trong viewport** cộng một ít buffer, thay vì render toàn bộ danh sách ra DOM. Khi cuộn, tái sử dụng node cho item mới vào tầm nhìn.

Vì sao cần: 10.000 row × mỗi row vài chục DOM node là hàng trăm nghìn node. Render lần đầu vài giây, cuộn giật, bộ nhớ lớn. Với virtualization, DOM chỉ có vài chục row bất kể list dài bao nhiêu.

Cách hoạt động: container có `height` cố định và `overflow: auto`; bên trong một div cao bằng tổng chiều cao thật để có scrollbar đúng; item được `position: absolute` với `top` tính từ index × row height. Theo dõi `scrollTop` để biết index nào đang hiện.

Dự án dùng `react-virtualized` cho danh sách khách hàng lớn. Thư viện hiện đại hơn: `react-window`, `@tanstack/react-virtual`.

Đánh đổi: khó với chiều cao item động (cần đo), Ctrl+F không tìm được item ngoài viewport, SEO không thấy toàn bộ.

**58. Socket.io lifecycle trong React?**

- **Tạo kết nối một lần**, không tạo lại mỗi render. Giữ instance ở module scope, trong `useRef`, hoặc trong Context/Redux để nhiều component dùng chung.
- **Đăng ký listener trong `useEffect`**, cleanup bằng `socket.off(event, handler)` với **cùng tham chiếu handler**. `socket.off(event)` không truyền handler sẽ gỡ mọi listener của event đó, có thể gỡ nhầm của component khác.
- **Disconnect khi unmount** component sở hữu kết nối (thường là app hoặc layout), không disconnect ở component con.
- Handler cập nhật state nên dùng functional update `setList(prev => [...prev, msg])` để không phụ thuộc state cũ, tránh phải đưa state vào dependency và re-subscribe liên tục.
- Kết hợp TanStack Query: nhận event thì `invalidateQueries` hoặc `setQueryData` thay vì giữ bản sao riêng.
- Xử lý reconnect: `socket.on('connect')` để join lại room, không giả định kết nối luôn sống.

Xem `src/hooks/useAppSocket.ts` trong repo.

**59. HTTP caching cơ bản?**

Hai cơ chế:
- **Freshness**: `Cache-Control: max-age=31536000, immutable` → browser dùng cache không hỏi server. `no-cache` → phải hỏi server trước khi dùng. `no-store` → không lưu.
- **Validation**: `ETag` / `Last-Modified`. Browser gửi `If-None-Match` / `If-Modified-Since`, server trả `304 Not Modified` không kèm body nếu chưa đổi.

Chiến lược cho SPA:
- File JS/CSS build có **hash trong tên** (`main.a1b2c3.js`) → cache vĩnh viễn. Nội dung đổi thì tên đổi, không sợ cũ.
- `index.html` → `no-cache`, để luôn lấy được tên file mới sau deploy.
- API response thường `no-store` hoặc để TanStack Query quản lý cache ở tầng app.

Hỏi xoáy: sau deploy user vẫn thấy bản cũ? Do `index.html` bị cache, hoặc service worker cũ. Kiểm tra header của HTML.

**60. Critical rendering path và cách tối ưu tải trang?**

Critical rendering path là chuỗi bước từ nhận HTML đến pixel đầu tiên (câu 51). Tối ưu là làm ngắn chuỗi này và giảm tài nguyên chặn render.

- **Giảm JS**: code splitting theo route, tree shaking, bỏ thư viện nặng (moment → dayjs), lazy load component ít dùng.
- **Không chặn parse**: `<script defer>` hoặc `type="module"`, CSS critical inline, font `display: swap`.
- **Ảnh**: định dạng WebP/AVIF, `loading="lazy"`, `srcset`, kích thước đúng, CDN transform.
- **Mạng**: nén Brotli/gzip, HTTP/2, CDN, preconnect tới API domain, cache header đúng (câu 59).
- **Runtime**: virtualization, debounce, memo hợp lý, tránh layout thrashing.
- **Đo**: Lighthouse, Web Vitals (LCP, INP, CLS), bundle analyzer để biết gì nặng.

Nói được cách đo quan trọng hơn liệt kê kỹ thuật.

---

## Nhóm 7. Tooling và build

**61. Vite là gì và khác Webpack ra sao?**

Vite là dev server + build tool.

**Dev**: phục vụ file nguồn dưới dạng **ES module native**, browser tự `import` từng file khi cần. Không bundle toàn app khi khởi động nên start gần tức thì bất kể dự án lớn. Dependency trong `node_modules` được pre-bundle một lần bằng esbuild (viết bằng Go, nhanh gấp 10–100 lần). HMR chỉ swap đúng module đổi.

**Build**: dùng Rollup bundle, minify, code split, hash tên file cho production.

**Webpack**: bundle toàn bộ đồ thị module cả khi dev, dự án lớn thì start và rebuild chậm. Đổi lại cấu hình linh hoạt và hệ sinh thái loader/plugin lâu năm.

Hỏi xoáy: vì sao dev và build dùng hai công cụ khác? esbuild cực nhanh nhưng chưa đủ tính năng code splitting và plugin cho production, Rollup mạnh hơn ở đó. Vite đang chuyển sang Rolldown để hợp nhất.

**62. Tree shaking là gì?**

Bundler phân tích tĩnh đồ thị `import`/`export`, xác định export nào không được dùng, loại khỏi bundle. Tên gọi: rung cây cho lá chết rơi.

Điều kiện để hoạt động:
- Dùng **ES module** (`import`/`export`) vì phân tích được lúc build. CommonJS `require()` là động, không shake được.
- Import cụ thể `import { debounce } from 'lodash-es'`, không `import _ from 'lodash'` (kéo cả thư viện).
- Module không có **side effect** ở top level, hoặc khai báo `"sideEffects": false` trong `package.json`. Bundler không dám bỏ module có thể chạy code khi import.
- Không dùng `export default { a, b }` object lớn rồi truy cập property, bundler không tách được.

Liên hệ dự án: `export const icons = { ... }` gom mọi icon vào một object, mọi icon đều vào bundle dù chỉ dùng vài cái. Đánh đổi có ý thức để tiện dùng.

**63. Code splitting là gì?**

Tách bundle thành nhiều **chunk** tải theo nhu cầu thay vì một file lớn. Bundle đầu nhỏ, trang tương tác nhanh hơn, code chưa cần thì chưa tải.

Ba cách tách:
- **Theo route**: `React.lazy(() => import('./CustomersPage'))`, phổ biến nhất.
- **Theo component nặng**: editor, chart, modal ít mở.
- **Vendor chunk**: tách `react`, `antd` ra file riêng để cache lâu, code app đổi không làm user tải lại vendor. Vite cấu hình qua `build.rollupOptions.output.manualChunks`.

Điểm cần lưu ý: quá nhiều chunk nhỏ tăng số request; preload chunk của route kế tiếp khi hover link; xử lý lỗi tải chunk cũ sau deploy bằng Error Boundary và reload.

**64. Env var trong Vite?**

- Chỉ biến có prefix **`VITE_`** được expose vào client qua `import.meta.env.VITE_API_URL`. Biến khác trong `.env` bị bỏ để tránh lộ nhầm.
- File theo mode: `.env`, `.env.development`, `.env.production`, `.env.local` (không commit). Mode chọn bằng `--mode`.
- Giá trị được **thay thế tĩnh lúc build**, nhúng thẳng vào bundle dưới dạng string. Không có runtime lookup. Đổi env phải build lại.
- Vì nằm trong bundle, **ai cũng đọc được**. Không đặt secret, API key riêng tư, connection string. Chỉ đặt URL công khai, feature flag, public key.
- Type cho env: khai báo `interface ImportMetaEnv` trong `vite-env.d.ts`.

Khác NestJS: BE đọc `process.env` lúc runtime, có thể đổi mà không build lại và không lộ ra ngoài.

**65. Feature flag ở FE làm sao?**

Bật/tắt tính năng mà không deploy code mới hoặc để release dần.

Nguồn flag:
- **Env var** `VITE_FEATURE_X=true`: đơn giản, nhưng đổi phải build lại. Dự án dùng cách này, ví dụ `VITE_SETTING_UNSUBSCRIBES_FLAG`.
- **API / remote config**: đổi realtime, bật theo venue/user, A/B test. Cần fetch lúc khởi động và lưu vào store.

Cách dùng: điều kiện render route, menu item, nút. Gom kiểm tra vào một hook `useFeatureFlag('x')` thay vì rải `import.meta.env` khắp nơi.

Quan trọng: flag FE chỉ là **UX gate**. Ẩn nút không ngăn user gọi API. BE vẫn phải kiểm tra quyền và flag riêng.

Dọn flag sau khi tính năng ổn định, tránh nợ kỹ thuật.

**66. Path alias `@/` là gì?**

Map `@/` về `src/` để import `@/components` thay vì `../../../components`.

Cấu hình hai nơi phải khớp:
- `tsconfig.json`: `"paths": { "@/*": ["./src/*"] }` cho TypeScript hiểu.
- `vite.config.ts`: `resolve.alias` cho bundler resolve. Thiếu một bên thì IDE ok nhưng build lỗi hoặc ngược lại.

Lợi ích: dễ đọc, di chuyển file không vỡ import, phân biệt rõ import ra ngoài module (`@/`) và trong module (`./`, `../`).

Quy ước dự án: ra khỏi `src/modules/[feature]/` thì dùng `@/`, trong module dùng relative. Thấy `../../` là dấu hiệu sai.

**67. ESLint và Prettier khác nhau thế nào?**

- **ESLint**: phân tích code tìm **lỗi và code smell**: biến không dùng, thiếu dependency trong `useEffect` (`react-hooks/exhaustive-deps`), gọi hook có điều kiện (`rules-of-hooks`), `any`, import vòng. Có thể `--fix` một phần. Cấu hình theo plugin: `@typescript-eslint`, `react-hooks`, `import`.
- **Prettier**: chỉ **format**: dấu chấm phẩy, ngoặc, xuống dòng, khoảng trắng. Không quan tâm logic. Opinionated, ít tùy chọn để tránh tranh cãi.

Dùng cả hai, tắt rule format của ESLint bằng `eslint-config-prettier` để không đá nhau. Chạy qua pre-commit hook (husky + lint-staged) và CI.

Rule quan trọng nhất với React: hai rule của `eslint-plugin-react-hooks`. Không bao giờ disable `exhaustive-deps` để cho qua.

**68. Test FE cơ bản dùng gì?**

- **Vitest**: test runner, API giống Jest, tích hợp Vite nên dùng chung alias và transform, nhanh.
- **React Testing Library (RTL)**: render component vào DOM giả (jsdom), query element theo cách user thấy: `getByRole('button', { name: /save/i })`, `getByText`, `getByLabelText`. Tránh query theo class hay test-id trừ khi bất khả kháng. Tương tác bằng `userEvent.click`, `userEvent.type`.
- **`renderHook`**: test custom hook độc lập, bọc Provider cần thiết (QueryClientProvider, i18n) qua `wrapper`.
- **Mock**: `vi.mock` module api, hoặc MSW chặn request ở tầng network cho gần thật hơn.
- **Async**: `await screen.findByText`, `waitFor` cho state cập nhật sau fetch.

Triết lý RTL: test **hành vi** (user bấm nút thì thấy toast) không test **implementation** (state có giá trị x). Refactor nội bộ không làm test đỏ.

Kim tự tháp: nhiều unit test cho helper và hook, vừa phải integration test cho component, ít E2E (Playwright/Cypress) cho luồng chính.

---

## Mẹo trả lời cho dev backend

- **So sánh với khái niệm BE quen thuộc**: query key giống cache key Redis; `useEffect` cleanup giống `onModuleDestroy`; Context giống DI scope cho một subtree; 3 lớp api/react-query/hooks giống Repository/Service/Controller; Yup giống class-validator; env `VITE_` khác `process.env` ở chỗ nhúng lúc build.
- **Nhấn mạnh điểm mạnh**: hiểu API contract, error handling, async, CORS, auth. Người phỏng vấn biết dev BE chuyển sang sẽ yếu CSS, họ tìm tư duy hệ thống.
- **Ba mảng dễ hụt cần ôn kỹ**: render lifecycle và `useEffect` (câu 12–15, 20), cache của TanStack Query (câu 30–34), CSS layout cơ bản (câu 44–46).
- **Khi không biết**: nói "tôi chưa dùng cái này nhưng theo tôi hiểu nó giải quyết vấn đề X, tương tự Y ở backend" tốt hơn im lặng hoặc bịa.
- **Đọc một luồng hoàn chỉnh trong repo trước khi phỏng vấn**: `src/api/customers` → `src/react-query/customers` → `src/modules/customers/hooks` → component. Đọc xong, tự trả lời lại 11 câu có dấu `*` bằng ví dụ thật từ code đó.
