# Bộ câu hỏi phỏng vấn Intern IT (Fullstack JS/TS)

Dựa trên tech stack: React.js, Next.js, Tailwind CSS, Ant Design, shadcn/ui, TanStack Query, Zustand, Axios, Zod, Express.js, NestJS, JWT, OAuth 2.0, Socket.IO, SSE, BullMQ, Swagger, AI Integration, SePay Integration, MongoDB, Supabase, Prisma ORM, Redis, Cloudinary, Git/GitHub.

Mỗi câu hỏi kèm gợi ý câu trả lời để đánh giá mức độ hiểu của ứng viên. Với intern, không cần đòi hỏi trả lời hoàn hảo — quan trọng là hiểu bản chất và biết cách tư duy khi gặp tình huống thực tế.

---

## 1. JavaScript / TypeScript

**Q1. Sự khác nhau giữa `let`, `const`, `var`?**
Trả lời: `var` có function scope, bị hoisting và có thể khai báo lại. `let`/`const` có block scope, không bị truy cập trước khi khai báo (temporal dead zone). `const` không cho gán lại tham chiếu, nhưng nếu là object/array thì vẫn có thể thay đổi thuộc tính bên trong.

**Q2. Giải thích Event Loop trong JavaScript.**
Trả lời: JS là single-threaded, dùng Event Loop để xử lý bất đồng bộ. Call stack chạy code đồng bộ; các tác vụ bất đồng bộ (setTimeout, Promise, I/O) được đẩy vào Callback Queue hoặc Microtask Queue. Microtask (Promise.then) luôn được ưu tiên xử lý trước Macrotask (setTimeout) sau mỗi lần call stack rỗng.

**Q3. `interface` và `type` trong TypeScript khác nhau ở điểm nào?**
Trả lời: Cả hai đều dùng để định nghĩa kiểu dữ liệu. `interface` có thể được "declaration merging" (khai báo nhiều lần rồi gộp lại) và thường dùng cho object/class. `type` linh hoạt hơn, có thể định nghĩa union, intersection, tuple, mapped type. Về hiệu năng compile gần như không khác biệt đáng kể.

**Q4. Generic trong TypeScript dùng để làm gì? Cho ví dụ.**
Trả lời: Generic giúp viết code tái sử dụng mà vẫn giữ được type-safety, thay vì dùng `any`. Ví dụ: `function getFirst<T>(arr: T[]): T { return arr[0]; }` — hàm này dùng được cho mọi kiểu mảng mà vẫn suy ra đúng kiểu trả về.

**Q5. `Promise.all` và `Promise.allSettled` khác nhau thế nào?**
Trả lời: `Promise.all` sẽ reject ngay khi có 1 promise fail, các kết quả khác bị bỏ qua. `Promise.allSettled` luôn đợi tất cả promise hoàn thành (dù fulfilled hay rejected) và trả về mảng kết quả kèm trạng thái từng cái — phù hợp khi cần biết kết quả của tất cả tác vụ dù có lỗi.

---

## 2. React.js & Next.js

**Q6. Sự khác biệt giữa Client Component và Server Component trong Next.js (App Router)?**
Trả lời: Server Component render trên server, không gửi JS xuống client, phù hợp cho phần tĩnh/lấy dữ liệu, giúp giảm bundle size. Client Component (`"use client"`) chạy trên trình duyệt, dùng khi cần state, hook, sự kiện tương tác (onClick, useEffect...).

**Q7. `useEffect` dùng để làm gì? Vì sao cần khai báo dependency array đúng?**
Trả lời: `useEffect` chạy side-effect (gọi API, subscribe, thao tác DOM) sau khi component render. Dependency array quyết định khi nào effect chạy lại; nếu thiếu dependency có thể gây bug logic cũ (stale closure), còn thừa dependency không cần thiết có thể gây re-run liên tục hoặc vòng lặp vô hạn.

**Q8. Vì sao dùng TanStack Query thay vì tự gọi API bằng Axios trong `useEffect`?**
Trả lời: TanStack Query tự động xử lý cache, refetch khi focus lại tab, retry khi lỗi, loading/error state, đồng bộ dữ liệu giữa nhiều component dùng chung 1 query key, tránh phải tự viết state quản lý thủ công như `isLoading`, `error`, cache.

**Q9. Zustand khác Redux ở điểm nào? Khi nào bạn chọn Zustand?**
Trả lời: Zustand gọn nhẹ hơn, không cần boilerplate (action, reducer, dispatch), API đơn giản dựa trên hook, không cần Provider bọc toàn app. Phù hợp cho state đơn giản đến trung bình. Redux mạnh hơn về middleware, devtools, quản lý state phức tạp ở dự án lớn.

**Q10. Vì sao cần validate dữ liệu ở cả frontend (Zod) lẫn backend, không chỉ 1 nơi?**
Trả lời: Validate ở frontend giúp UX tốt (báo lỗi ngay, không cần gọi API). Nhưng frontend có thể bị bypass (gọi API trực tiếp qua Postman/curl), nên backend luôn phải validate lại để đảm bảo bảo mật và toàn vẹn dữ liệu.

**Q11. SSR, SSG, ISR trong Next.js khác nhau ra sao?**
Trả lời: SSR (Server-Side Rendering) render mỗi request. SSG (Static Site Generation) render sẵn lúc build, phục vụ file tĩnh — nhanh nhưng dữ liệu không cập nhật real-time. ISR (Incremental Static Regeneration) là SSG nhưng có thể revalidate lại sau 1 khoảng thời gian mà không cần build lại toàn bộ site.

---

## 3. UI Libraries (Tailwind, Ant Design, shadcn/ui)

**Q12. Tailwind CSS khác gì so với viết CSS thuần hoặc dùng thư viện component như Ant Design?**
Trả lời: Tailwind là utility-first, viết class trực tiếp trong HTML/JSX thay vì viết file CSS riêng, giúp code nhanh và tránh CSS thừa (dùng PurgeCSS loại bỏ class không dùng). Ant Design là bộ component có sẵn UI/logic hoàn chỉnh (Table, Form...), phù hợp làm nhanh nhưng khó tùy biến sâu. shadcn/ui thì cung cấp source code component (copy vào project) để tự chỉnh sửa thoải mái, kết hợp Tailwind.

**Q13. Khi nào bạn chọn dùng Ant Design, khi nào chọn shadcn/ui cho 1 dự án mới?**
Trả lời: Ant Design phù hợp dự án cần ra sản phẩm nhanh, nhiều component phức tạp có sẵn (bảng dữ liệu, form phức tạp), chấp nhận UI theo phong cách chuẩn của Ant. shadcn/ui phù hợp khi cần thiết kế UI riêng, đồng bộ với design system của công ty, cần custom sâu.

---

## 4. Backend (Express.js, NestJS)

**Q14. Middleware trong Express.js là gì?**
Trả lời: Middleware là hàm chạy giữa lúc request đến và response trả về, có thể xử lý logic như xác thực, log, validate, hoặc chỉnh sửa request/response trước khi đến route handler tiếp theo, thông qua gọi `next()`.

**Q15. NestJS khác Express.js như thế nào?**
Trả lời: Express là framework tối giản, tự do tổ chức code. NestJS xây trên Express (hoặc Fastify), có kiến trúc rõ ràng theo module - controller - service, hỗ trợ Dependency Injection, decorator, dễ maintain và test hơn cho dự án lớn.

**Q16. Dependency Injection trong NestJS là gì và lợi ích của nó?**
Trả lời: DI là cách NestJS tự động khởi tạo và "tiêm" các dependency (service, repository...) vào class cần dùng thay vì tự new thủ công. Lợi ích: giảm sự phụ thuộc cứng giữa các class, dễ viết unit test (mock dependency), dễ quản lý vòng đời object.

---

## 5. Xác thực & bảo mật (JWT, OAuth 2.0)

**Q17. JWT gồm những phần nào? Vì sao không nên lưu dữ liệu nhạy cảm trong payload?**
Trả lời: JWT gồm 3 phần: Header, Payload, Signature. Payload chỉ được encode Base64 chứ không mã hóa, nên ai cũng decode đọc được nội dung — vì vậy không được để password hay dữ liệu nhạy cảm trong đó.

**Q18. Access Token và Refresh Token khác nhau thế nào? Tại sao cần cả hai?**
Trả lời: Access Token có thời hạn ngắn, dùng để xác thực mỗi request. Refresh Token có thời hạn dài hơn, dùng để cấp lại Access Token mới khi hết hạn mà không bắt user đăng nhập lại. Tách 2 loại giúp giảm rủi ro nếu Access Token bị lộ (thời gian sống ngắn) trong khi vẫn giữ trải nghiệm đăng nhập lâu dài.

**Q19. Tình huống: Access token của user bị đánh cắp (lộ ra ngoài). Hệ thống dùng JWT thuần (stateless) thì xử lý sao vì JWT không thể "hủy" giữa chừng?**
Trả lời: Vì JWT là stateless nên không thể revoke trực tiếp. Cách xử lý: để thời gian sống Access Token thật ngắn (5-15 phút); dùng danh sách blacklist (lưu trong Redis) cho các token/refresh token bị thu hồi và kiểm tra danh sách này mỗi request; hoặc lưu 1 "version"/"session id" trong DB user, khi cần revoke thì tăng version lên, mọi token cũ có version thấp hơn coi như không hợp lệ.

**Q20. OAuth 2.0 dùng để làm gì? Phân biệt với JWT.**
Trả lời: OAuth 2.0 là giao thức ủy quyền (authorization), cho phép ứng dụng truy cập tài nguyên user ở bên thứ ba (VD: đăng nhập bằng Google) mà không cần biết mật khẩu user. JWT chỉ là định dạng token, có thể được dùng làm access token trong luồng OAuth, nhưng bản thân JWT không phải là 1 giao thức xác thực/ủy quyền.

---

## 6. Realtime (Socket.IO, SSE)

**Q21. Socket.IO khác gì WebSocket thuần?**
Trả lời: Socket.IO xây trên WebSocket nhưng có thêm cơ chế fallback (long-polling) khi WebSocket không khả dụng, tự động reconnect, hỗ trợ room/namespace để nhóm client, và có event-based API dễ dùng hơn WebSocket thuần.

**Q22. SSE (Server-Sent Events) khác Socket.IO/WebSocket như thế nào? Khi nào dùng SSE thay vì WebSocket?**
Trả lời: SSE chỉ truyền dữ liệu một chiều từ server xuống client (qua HTTP), đơn giản, tự động reconnect, phù hợp cho các trường hợp chỉ cần server đẩy dữ liệu (thông báo, tiến trình AI đang generate, live feed). WebSocket/Socket.IO thì hai chiều (client cũng gửi được lên server liên tục), phù hợp cho chat, game realtime.

**Q23. Tình huống: Ứng dụng chat dùng Socket.IO, khi scale lên nhiều server (nhiều instance) thì user ở server A gửi tin không tới được user đang connect ở server B. Xử lý sao?**
Trả lời: Vì mỗi server chỉ giữ kết nối socket riêng, cần một lớp trung gian để đồng bộ event giữa các instance — thường dùng Socket.IO Redis Adapter. Redis đóng vai trò pub/sub: khi 1 server emit event, nó publish lên Redis, các server khác subscribe và emit lại tới đúng client của mình.

---

## 7. Redis

**Q24. Redis dùng để làm gì trong hệ thống? Cho ví dụ.**
Trả lời: Redis là in-memory data store, thường dùng để cache dữ liệu hay truy vấn (giảm tải DB), lưu session, làm message broker (pub/sub), hàng đợi (queue), rate limiting, hoặc lưu OTP/token có thời gian sống ngắn (TTL).

**Q25. Tình huống: Có 1 API đọc dữ liệu nặng (query DB tốn thời gian) và bạn đã cache kết quả vào Redis. Vào 1 thời điểm cache hết hạn (expire), có 1000 user cùng lúc gọi API này — chuyện gì xảy ra và cách xử lý?**
Trả lời: Đây là hiện tượng "Cache Stampede" (hay "Thundering Herd") — vì cache đã hết hạn, cả 1000 request đều miss cache và cùng dồn vào query DB trực tiếp, có thể làm DB quá tải hoặc sập. Cách xử lý:
- Dùng cơ chế lock (mutex) trong Redis: chỉ 1 request được phép query DB và set lại cache, các request khác đợi hoặc trả kết quả cache cũ (stale) trong lúc chờ.
- Set TTL có random offset (jitter) để cache không hết hạn cùng lúc.
- Dùng kỹ thuật "stale-while-revalidate": vẫn trả dữ liệu cũ trong khi 1 tiến trình nền âm thầm refresh cache mới.
- Cache Warming: chủ động refresh cache trước khi hết hạn thay vì để tự hết hạn.

**Q26. Redis có những kiểu dữ liệu nào? Kể ra vài loại và ứng dụng thực tế.**
Trả lời: String (cache giá trị đơn giản, đếm số), Hash (lưu object như thông tin user), List (queue đơn giản, feed), Set (danh sách không trùng, ví dụ tag), Sorted Set (bảng xếp hạng - leaderboard vì có điểm số để sắp xếp).

---

## 8. BullMQ (Job Queue)

**Q27. BullMQ dùng để làm gì? Cho ví dụ tình huống thực tế.**
Trả lời: BullMQ là thư viện quản lý hàng đợi job (dựa trên Redis), dùng để xử lý các tác vụ nặng/chậm ở background thay vì chặn request chính. Ví dụ: gửi email hàng loạt, xử lý resize ảnh, gọi AI generate nội dung, tạo báo cáo PDF.

**Q28. Tình huống: Một job trong BullMQ xử lý gửi email bị lỗi (email service down tạm thời). Làm sao đảm bảo job không bị mất và được xử lý lại?**
Trả lời: BullMQ hỗ trợ cấu hình `attempts` (số lần thử lại) và `backoff` (thời gian chờ giữa các lần retry, có thể tăng dần - exponential backoff) khi tạo job. Nếu vẫn fail sau số lần retry tối đa, job sẽ vào trạng thái "failed" và có thể được lưu lại để xử lý thủ công hoặc đẩy sang queue riêng (dead-letter queue) để không bị mất.

**Q29. Vì sao nên xử lý job nặng bằng queue thay vì xử lý trực tiếp trong request API?**
Trả lời: Nếu xử lý trực tiếp, user phải đợi lâu (request timeout), và nếu nhiều request cùng lúc gọi tác vụ nặng sẽ làm server quá tải. Dùng queue giúp trả response ngay cho user (VD: "đang xử lý"), còn việc nặng chạy nền, có thể scale worker riêng, và dễ retry khi lỗi.

---

## 9. Database (MongoDB, Supabase, Prisma ORM)

**Q30. MongoDB khác gì với SQL Database (như PostgreSQL)?**
Trả lời: MongoDB là NoSQL, lưu dữ liệu dạng document (giống JSON), schema linh hoạt, không bắt buộc join phức tạp — phù hợp dữ liệu thay đổi cấu trúc thường xuyên. SQL Database có schema cố định, hỗ trợ transaction và ràng buộc quan hệ (foreign key) chặt chẽ hơn, phù hợp dữ liệu có cấu trúc rõ ràng, cần tính toàn vẹn cao.

**Q31. Prisma ORM là gì? Vì sao nên dùng ORM thay vì viết raw SQL/query trực tiếp?**
Trả lời: Prisma là ORM (Object-Relational Mapping) giúp thao tác với DB bằng code JS/TS thay vì viết câu query thủ công, tự sinh type-safe client dựa theo schema. Lợi ích: tránh lỗi cú pháp SQL, gợi ý type ngay trong code, dễ migrate schema, hạn chế SQL Injection vì Prisma tự escape input.

**Q32. Supabase là gì?**
Trả lời: Supabase là nền tảng Backend-as-a-Service mã nguồn mở, dùng PostgreSQL làm nền, cung cấp sẵn Authentication, Realtime subscription, Storage, và tự sinh API (REST/GraphQL) từ database — giúp làm backend nhanh mà không cần tự dựng server từ đầu.

**Q33. Tình huống: 2 user cùng lúc đặt mua sản phẩm cuối cùng trong kho (chỉ còn 1 sản phẩm). Làm sao tránh bị bán trùng (race condition)?**
Trả lời: Cần đảm bảo thao tác kiểm tra tồn kho và trừ kho diễn ra nguyên tử (atomic), tránh 2 request đọc cùng giá trị rồi cùng ghi đè. Cách xử lý phổ biến:
- Dùng transaction của DB kết hợp câu lệnh update có điều kiện (VD: `UPDATE products SET stock = stock - 1 WHERE id = ? AND stock > 0`), chỉ update thành công nếu còn hàng.
- Hoặc dùng lock ở tầng ứng dụng (Redis distributed lock) để đảm bảo chỉ 1 request xử lý tại 1 thời điểm cho cùng 1 sản phẩm.
- Với MongoDB có thể dùng `findOneAndUpdate` với điều kiện tương tự để đảm bảo tính nguyên tử.

---

## 10. Cloudinary, Swagger, AI Integration, SePay

**Q34. Vì sao nên dùng dịch vụ như Cloudinary để lưu ảnh thay vì lưu trực tiếp trên server?**
Trả lời: Lưu ảnh trực tiếp trên server tốn dung lượng, khó scale khi có nhiều instance (server này không thấy file server kia lưu), không có CDN nên load chậm. Cloudinary cung cấp CDN toàn cầu, tự động resize/optimize ảnh, và tách biệt việc lưu trữ file khỏi server backend giúp hệ thống dễ scale hơn.

**Q35. Swagger dùng để làm gì trong 1 dự án backend?**
Trả lời: Swagger (OpenAPI) dùng để tự động sinh tài liệu API (endpoint, params, request/response mẫu), giúp frontend hoặc bên thứ ba biết cách gọi API mà không cần hỏi trực tiếp dev backend, đồng thời có thể test API ngay trên giao diện Swagger UI.

**Q36. Khi tích hợp AI (VD: gọi API của OpenAI/Anthropic) vào hệ thống, bạn cần lưu ý gì về hiệu năng và chi phí?**
Trả lời: Cần lưu ý: response từ AI có thể mất vài giây nên nên xử lý bất đồng bộ (queue) hoặc dùng streaming (SSE) để trả dữ liệu dần cho user thay vì bắt đợi toàn bộ; cần giới hạn rate limit/số request để tránh chi phí phát sinh quá lớn; nên cache lại các câu trả lời cho câu hỏi lặp lại nếu phù hợp; và cần xử lý lỗi khi AI service bị timeout hoặc trả về không đúng định dạng mong đợi.

**Q37. Tích hợp cổng thanh toán (như SePay) cần lưu ý gì về mặt bảo mật và độ tin cậy?**
Trả lời: Cần xác thực webhook (chữ ký/signature) từ SePay gửi về để đảm bảo request thật sự đến từ SePay chứ không phải giả mạo; xử lý idempotent (nếu webhook bị gửi lặp lại do lỗi mạng thì không được cộng tiền/xác nhận đơn hàng 2 lần); lưu log giao dịch để đối soát; và không nên tin tưởng hoàn toàn kết quả trả về ngay lúc frontend gọi API mà cần đợi xác nhận qua webhook từ server thanh toán.

---

## 11. Git & Github

**Q38. `git merge` và `git rebase` khác nhau thế nào?**
Trả lời: `git merge` gộp 2 nhánh lại, tạo thêm 1 merge commit, giữ nguyên lịch sử commit của cả 2 nhánh. `git rebase` viết lại lịch sử bằng cách "chuyển" các commit của nhánh hiện tại lên trên đầu nhánh đích, tạo lịch sử thẳng hàng, dễ đọc hơn nhưng thay đổi commit hash gốc — không nên rebase nhánh đã push và có người khác đang dùng chung.

**Q39. Tình huống: Bạn và đồng nghiệp cùng sửa 1 file, khi pull code về bị conflict. Bạn xử lý thế nào?**
Trả lời: Mở file bị conflict, Git sẽ đánh dấu đoạn xung đột bằng `<<<<<<<`, `=======`, `>>>>>>>`. Cần đọc hiểu cả 2 phần thay đổi, trao đổi với đồng nghiệp nếu cần, chọn giữ lại đúng logic mong muốn (có thể giữ cả 2, sửa lại cho hợp lý), xóa các ký hiệu đánh dấu, sau đó `git add` file đã resolve và commit lại.

---

## 12. Câu hỏi tổng hợp về tư duy hệ thống

**Q40. Tình huống: Website của bạn đột nhiên load rất chậm vào giờ cao điểm. Bạn sẽ kiểm tra và xử lý theo hướng nào?**
Trả lời: Trả lời tốt cần thể hiện quy trình debug có hệ thống: kiểm tra log/monitoring xem bottleneck ở đâu (server CPU/RAM, DB query chậm, hay network); kiểm tra DB có query thiếu index không; kiểm tra có đang bị N+1 query không; xem có thể thêm cache (Redis) cho dữ liệu ít thay đổi; xem xét đưa tác vụ nặng ra background queue (BullMQ); và cân nhắc scale ngang (thêm instance) nếu do lượng truy cập quá lớn.

**Q41. Tình huống: User phản ánh rằng đôi khi họ bị đăng xuất đột ngột dù chưa hết phiên làm việc. Bạn sẽ debug theo hướng nào?**
Trả lời: Kiểm tra thời gian sống (expiry) của access token và refresh token có hợp lý không; kiểm tra cơ chế tự động refresh token ở frontend có hoạt động đúng không (VD: interceptor của Axios bắt lỗi 401 để tự gọi refresh); kiểm tra đồng hồ server và client có lệch giờ không (ảnh hưởng việc kiểm tra `exp` của JWT); và kiểm tra xem có phải do nhiều tab/nhiều thiết bị đăng nhập cùng lúc làm token cũ bị vô hiệu hóa không.

---

## 13. Thêm câu hỏi xử lý tình huống (mở rộng)

### Frontend (React, Next.js, TanStack Query, Zustand, Axios)

**Q42. Tình huống: Trang danh sách sản phẩm bị re-render liên tục mỗi khi gõ vào ô tìm kiếm, làm UI giật lag. Bạn sẽ xử lý thế nào?**
Trả lời: Trước tiên xác định nguyên nhân re-render (dùng React DevTools Profiler). Các hướng xử lý: debounce/throttle input tìm kiếm để không gọi setState/API liên tục theo từng ký tự; dùng `React.memo` cho các component con không cần re-render theo state search; tách state search ra khỏi component cha lớn để tránh kéo theo re-render toàn bộ cây con; dùng `useMemo`/`useCallback` hợp lý để tránh tạo lại object/hàm mới mỗi lần render.

**Q43. Tình huống: Dùng TanStack Query để lấy dữ liệu giỏ hàng, nhưng sau khi user thêm sản phẩm mới, danh sách hiển thị không cập nhật ngay dù API thêm đã thành công. Vì sao và cách xử lý?**
Trả lời: Vì TanStack Query cache dữ liệu theo `queryKey` và không tự biết dữ liệu đã đổi sau khi gọi mutation. Cần dùng `queryClient.invalidateQueries({ queryKey: [...] })` sau khi mutation thành công để buộc query refetch lại, hoặc dùng `onSuccess`/`optimistic update` để cập nhật cache ngay lập tức mà không cần đợi gọi lại API, giúp UX mượt hơn.

**Q44. Tình huống: Nhiều nơi trong app gọi API bị lỗi 401 (token hết hạn) và mỗi nơi tự xử lý logout khác nhau, gây trải nghiệm không nhất quán. Bạn thiết kế lại thế nào với Axios?**
Trả lời: Tạo Axios instance dùng chung và cấu hình interceptor response tập trung: khi gặp lỗi 401, tự động gọi API refresh token; nếu refresh thành công thì retry lại request cũ; nếu refresh thất bại thì mới thực hiện logout và redirect về trang đăng nhập — xử lý logic này một lần duy nhất ở interceptor thay vì rải rác nhiều nơi.

**Q45. Tình huống: State giỏ hàng lưu trong Zustand, nhưng khi user mở 2 tab trình duyệt cùng lúc, thêm sản phẩm ở tab A thì tab B không thấy cập nhật. Có cần xử lý không, và xử lý thế nào nếu cần?**
Trả lời: Zustand mặc định chỉ lưu state trong bộ nhớ của tab đó nên các tab không tự đồng bộ với nhau. Nếu nghiệp vụ yêu cầu đồng bộ giữa các tab, có thể kết hợp `localStorage` + sự kiện `storage` (browser tự bắn event này khi tab khác thay đổi localStorage) để các tab lắng nghe và cập nhật lại state Zustand tương ứng, hoặc dùng BroadcastChannel API.

**Q46. Tình huống: Form dùng Zod để validate, nhưng có 1 field cần validate bất đồng bộ (VD: kiểm tra email đã tồn tại trong hệ thống chưa qua API). Zod xử lý được không?**
Trả lời: Zod hỗ trợ validate bất đồng bộ thông qua `.refine()` hoặc `.superRefine()` với hàm callback trả về Promise, kết hợp gọi `schema.parseAsync()` thay vì `parse()`. Tuy nhiên cần lưu ý gọi API kiểm tra dạng này nên debounce để tránh gọi liên tục khi user đang gõ.

### Backend (Express, NestJS)

**Q47. Tình huống: API tạo đơn hàng chạy đúng lúc test nhưng khi lên production, thỉnh thoảng đơn hàng bị tạo thiếu 1 vài trường dữ liệu (do một service phụ như gửi email hoặc trừ kho bị lỗi giữa chừng). Bạn thiết kế lại logic thế nào?**
Trả lời: Vấn đề là thao tác tạo đơn hàng gồm nhiều bước (lưu đơn, trừ kho, gửi email...) nhưng không đảm bảo tính toàn vẹn (atomicity). Cần bọc các thao tác ghi dữ liệu quan trọng (tạo đơn + trừ kho) trong 1 transaction của DB — nếu bước nào lỗi thì rollback toàn bộ. Các tác vụ không bắt buộc phải đồng bộ ngay (gửi email, thông báo) nên tách ra xử lý qua queue (BullMQ) sau khi transaction chính đã thành công, tránh làm hỏng luồng chính nếu service phụ bị lỗi.

**Q48. Tình huống: Một middleware xác thực (authentication) trong Express được đặt sau route xử lý thay vì trước. Hậu quả là gì?**
Trả lời: Middleware trong Express chạy tuần tự theo thứ tự khai báo. Nếu middleware xác thực đặt sau route, thì request sẽ đi thẳng vào route xử lý trước khi được kiểm tra token — nghĩa là API đó không được bảo vệ, ai cũng gọi được kể cả chưa đăng nhập. Cần đảm bảo middleware xác thực đăng ký trước các route cần bảo vệ.

**Q49. Tình huống: Bạn cần giới hạn 1 API chỉ cho phép gọi tối đa 5 lần/phút cho mỗi user để tránh spam. Thiết kế giải pháp thế nào?**
Trả lời: Có thể triển khai rate limiting dùng Redis: mỗi lần user gọi API, tăng 1 counter theo key (VD: `rate:userId:endpoint`) và set TTL 60 giây; nếu counter vượt quá 5 trong khoảng TTL đó thì trả lỗi 429 (Too Many Requests). Ở NestJS có thể dùng Guard kết hợp thư viện như `@nestjs/throttler`, còn Express có thể dùng middleware như `express-rate-limit` kết hợp Redis store để hoạt động đúng khi có nhiều instance server.

**Q50. Tình huống: NestJS service của bạn cần gọi sang 1 microservice khác, nhưng service đó đôi khi phản hồi rất chậm hoặc down, làm ảnh hưởng cả hệ thống chính (cascading failure). Xử lý sao?**
Trả lời: Áp dụng các kỹ thuật resilience: đặt timeout hợp lý cho request gọi sang service khác (không để chờ vô hạn); dùng circuit breaker (VD: thư viện `opossum`) để tạm ngắt gọi sang service đang lỗi trong 1 khoảng thời gian, tránh dồn request làm nó sập nặng hơn; có cơ chế fallback (trả dữ liệu mặc định/cache) khi service phụ không phản hồi được, để không làm crash toàn bộ luồng chính.

### Database (MongoDB, Prisma, Supabase)

**Q51. Tình huống: Một collection MongoDB có hàng triệu bản ghi, câu query lọc theo `email` chạy rất chậm. Nguyên nhân và cách khắc phục?**
Trả lời: Nhiều khả năng do thiếu index trên trường `email`, khiến MongoDB phải quét toàn bộ collection (collection scan) thay vì tra cứu nhanh qua index. Cách khắc phục: tạo index cho trường `email` (`db.collection.createIndex({ email: 1 })`), và nếu trường đó cần là duy nhất thì tạo unique index. Cũng nên dùng `explain()` để phân tích query xem có đang dùng index đúng cách không.

**Q52. Tình huống: Sau khi deploy migration mới bằng Prisma lên production, một cột dữ liệu quan trọng vô tình bị xóa mất dữ liệu cũ. Làm sao tránh tình huống này trong tương lai?**
Trả lời: Cần có quy trình review migration trước khi apply lên production (không tự động chạy `prisma migrate deploy` mà chưa review file SQL sinh ra); luôn backup database trước khi chạy migration lớn; test migration trên môi trường staging giống production trước; với các thay đổi có nguy cơ mất dữ liệu (đổi tên cột, xóa cột), nên chia làm nhiều bước an toàn hơn (VD: thêm cột mới, migrate dữ liệu, sau đó mới xóa cột cũ ở lần deploy sau).

**Q53. Tình huống: Dùng Supabase, bạn phát hiện user A có thể xem được dữ liệu đơn hàng của user B qua API dù không có quyền. Nguyên nhân thường gặp là gì?**
Trả lời: Thường do chưa cấu hình đúng Row Level Security (RLS) trong Supabase/PostgreSQL — nếu bảng không bật RLS hoặc policy cho phép quá rộng, thì bất kỳ ai có API key (kể cả anon key) cũng có thể truy vấn dữ liệu của người khác. Cần bật RLS cho các bảng nhạy cảm và viết policy rõ ràng, VD: chỉ cho phép `SELECT` khi `auth.uid() = user_id`.

### Redis, Queue, Realtime (mở rộng)

**Q54. Tình huống: Redis server dùng để cache đột nhiên bị đầy bộ nhớ (out of memory). Nguyên nhân và cách xử lý?**
Trả lời: Có thể do cache các key không đặt TTL (tồn tại vĩnh viễn) khiến bộ nhớ tăng dần theo thời gian, hoặc cache dữ liệu quá lớn không cần thiết. Cách xử lý: luôn set TTL hợp lý cho key cache; cấu hình chính sách xóa bớt (eviction policy) như `allkeys-lru` để Redis tự loại bỏ các key ít dùng nhất khi đầy bộ nhớ; theo dõi (monitor) dung lượng Redis định kỳ; và cân nhắc tăng RAM hoặc tách cache theo nhiều instance nếu lượng dữ liệu cache thực sự lớn.

**Q55. Tình huống: Trong BullMQ, một job bị treo mãi ở trạng thái "active" (không chạy xong cũng không fail) do worker bị crash giữa chừng lúc đang xử lý. Làm sao phát hiện và xử lý các job "mồ côi" này?**
Trả lời: BullMQ có cơ chế "stalled job" — nếu worker không gửi tín hiệu (heartbeat/lock renewal) trong khoảng thời gian quy định, job sẽ được coi là stalled và tự động được đưa trở lại hàng đợi để worker khác (hoặc chính worker đó sau khi restart) xử lý lại, dựa trên cấu hình `stalledInterval` và `maxStalledCount`. Cần đảm bảo job có tính idempotent (chạy lại nhiều lần không gây lỗi dữ liệu, ví dụ không cộng tiền 2 lần) vì job có thể được thử lại.

**Q56. Tình huống: Client dùng SSE để nhận thông báo real-time, nhưng khi mất mạng vài giây rồi có lại, client không nhận được các thông báo đã bị gửi trong lúc mất kết nối. Xử lý sao?**
Trả lời: SSE hỗ trợ sẵn cơ chế tự động reconnect của trình duyệt, và cho phép server gửi kèm `id` cho mỗi sự kiện. Khi client reconnect, trình duyệt tự động gửi lại header `Last-Event-ID` chứa id cuối cùng đã nhận. Server cần lưu lại lịch sử event gần đây (VD: trong Redis) và khi thấy `Last-Event-ID`, gửi bù lại các event bị lỡ kể từ id đó trở đi.

**Q57. Tình huống: Ứng dụng chat Socket.IO, user báo rằng thỉnh thoảng bị nhận tin nhắn 2 lần giống nhau. Nguyên nhân có thể là gì?**
Trả lời: Thường do: client bị kết nối trùng (VD: mở nhiều tab hoặc reconnect nhưng không disconnect socket cũ trước đó, dẫn tới nhiều socket cùng lắng nghe 1 event); hoặc do đăng ký lại event listener nhiều lần (VD: gọi `socket.on()` trong component React mà không cleanup ở `useEffect`, khiến mỗi lần re-render lại thêm 1 listener mới). Cách xử lý: đảm bảo cleanup listener đúng cách (`socket.off()` khi unmount), và ở server có thể kiểm tra/đóng các kết nối cũ khi user kết nối lại.

### DevOps / Vận hành chung

**Q58. Tình huống: Sau khi deploy bản mới lên production, hệ thống bắt đầu lỗi hàng loạt. Bạn xử lý theo quy trình nào?**
Trả lời: Trước tiên ưu tiên khôi phục dịch vụ: rollback về bản deploy trước đó ổn định (nếu có sẵn cơ chế rollback nhanh) thay vì cố sửa lỗi ngay trên production đang gặp sự cố. Sau khi hệ thống ổn định lại, mới kiểm tra log, so sánh thay đổi trong bản deploy lỗi để xác định nguyên nhân gốc rễ, viết fix, test kỹ ở môi trường staging trước khi deploy lại.

**Q59. Tình huống: Bạn merge nhầm code chưa hoàn thiện vào nhánh `main` đang chạy production. Bạn xử lý ra sao mà không làm gián đoạn công việc của team?**
Trả lời: Có thể dùng `git revert` để tạo 1 commit mới đảo ngược thay đổi đó (an toàn hơn vì không làm mất lịch sử, không ảnh hưởng người khác đang pull code), thay vì dùng `git reset` trực tiếp trên nhánh chung (dễ gây mất commit của người khác nếu họ đã pull). Sau khi revert xong, sửa lại code ở nhánh riêng, test kỹ rồi mới merge lại.

**Q60. Tình huống: Bạn được giao 1 tính năng mới nhưng không chắc cách tiếp cận tối ưu, deadline gấp. Bạn sẽ làm gì?**
Trả lời: Đây là câu hỏi đánh giá thái độ làm việc, không có đáp án kỹ thuật cố định. Câu trả lời tốt thường thể hiện: chủ động trao đổi sớm với mentor/leader để làm rõ yêu cầu và xin ý kiến hướng tiếp cận thay vì tự đoán và làm sai hướng; ước lượng thời gian trung thực, báo sớm nếu thấy deadline không khả thi; ưu tiên làm đúng phần lõi (core) trước, các phần phụ có thể làm sau nếu thiếu thời gian; và luôn hỏi lại khi chưa chắc chắn thay vì im lặng làm sai.

---

*Gợi ý sử dụng: Với intern, nên ưu tiên các câu hỏi nền tảng (mục 1, 2, 6, 7, 9) và khoảng 4-6 câu tình huống phù hợp với vị trí (mục 13) để đánh giá tư duy giải quyết vấn đề, thay vì hỏi hết toàn bộ danh sách trong 1 buổi phỏng vấn.*
