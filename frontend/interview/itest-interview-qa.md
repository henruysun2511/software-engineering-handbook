# Câu hỏi & Trả lời phỏng vấn — iTEST
> Dựa trên code thực tế: useExamStore.ts, useAuthStore.ts, axios.ts, exam-editor.tsx, part-editor.tsx, group-editor.tsx, question-editor.tsx, media-render.tsx, media-uploader.tsx, useExamQuery.ts

---

## PHẦN 1 — STATE MANAGEMENT (Zustand)

---

### Câu 1: Em dùng Zustand để quản lý state cho Exam Editor có cấu trúc lồng nhau 3 cấp (Part → QuestionGroup → Question). Em thiết kế store như thế nào?

**Trả lời:**

Thực ra trong iTEST, em không đưa toàn bộ cây 3 cấp vào Zustand. Em dùng Zustand rất tối giản — `useExamStore` chỉ lưu `examData` (dữ liệu đề thi khi thí sinh vào thi) và `examResult` (kết quả sau khi nộp). Store này có đúng 2 setters là `setExamData` và `setExamResult`.

Còn toàn bộ logic chỉnh sửa editor (Part → QuestionGroup → Question) em quản lý bằng **local state `useState`** ở component cha, truyền xuống thông qua props và callback. Cụ thể:

- `ExamData` (gồm mảng `parts`) là controlled state ở page component
- Mỗi lần giáo viên sửa gì, component con gọi callback `updatePart(index, updatedPart)` truyền lên
- Component cha clone mảng, thay thế phần tử đúng index, rồi `setState` toàn bộ

Ví dụ trong `exam-editor.tsx`:
```ts
const updatePart = (index: number, updatedPart: any) => {
  const newParts = [...value.parts];
  newParts[index] = updatedPart;
  onChange({ ...value, parts: newParts });
};
```

**Lý do không nhét vào Zustand:** Editor là một form tạm thời, chỉ cần tồn tại trong session làm việc. Zustand phù hợp hơn cho global state cần share giữa nhiều route (như `examData` cần dùng ở cả trang thi lẫn trang kết quả). Nhét toàn bộ editor tree vào store sẽ phức tạp hóa không cần thiết và khó debug hơn.

---

### Câu 2: Khi giáo viên chỉnh sửa một Question, cả cây state có bị re-render không?

**Trả lời:**

Có, đây là trade-off của pattern "immutable update + props drilling" mà em đang dùng. Mỗi lần `updateQuestion` được gọi, nó gọi lên `updatePart` ở `PartEditor`, tiếp tục bubble lên `ExamEditor`, trigger re-render toàn bộ cây.

Cụ thể trong `question-editor.tsx`:
```ts
const updateQuestion = (index: number, updated: any) => {
  const newQuestions = [...part.questions];
  newQuestions[index] = updated;
  updatePart(partIndex, { ...part, questions: newQuestions });
};
```

Mỗi lần sửa 1 câu, `part` object mới được tạo ra, trigger re-render `PartEditor` và tất cả `QuestionEditor` bên trong.

**Tại sao vẫn chấp nhận được:** Vì đây là editor admin/teacher, thường chỉ có vài chục câu hỏi, không phải danh sách hàng nghìn item. React reconciliation đủ nhanh ở scale này. Nếu cần optimize thêm, em có thể wrap từng `QuestionEditor` bằng `React.memo` và đảm bảo callback không bị recreate (dùng `useCallback`).

---

### Câu 3: Tại sao `useAuthStore` dùng `persist` middleware còn `useExamStore` thì không?

**Trả lời:**

Vì hai store phục vụ mục đích hoàn toàn khác nhau về vòng đời dữ liệu.

`useAuthStore` cần persist vì:
- `user` và `accessToken` phải tồn tại qua browser reload
- Nếu không persist, user phải đăng nhập lại mỗi lần F5

```ts
export const useAuthStore = create<AuthState>()(
  persist(
    (set) => ({ user: null, accessToken: null, ... }),
    {
      name: "auth-storage",
      partialize: (state) => ({ user: state.user, accessToken: state.accessToken }),
    }
  )
);
```

Em còn dùng `partialize` để chỉ lưu `user` và `accessToken` vào localStorage, không lưu `isHydrated` (vì đó là runtime flag, không cần persist).

`useExamStore` không cần persist vì:
- `examData` chỉ cần tồn tại trong session thi, không cần sống qua reload
- Nếu reload giữa bài thi, thí sinh cần fetch lại từ server (có draft recovery từ Redis)
- Lưu exam data vào localStorage sẽ tốn bộ nhớ và có nguy cơ stale data

---

### Câu 4: `isHydrated` trong `useAuthStore` dùng để làm gì?

**Trả lời:**

`isHydrated` là flag để biết Zustand đã đọc xong data từ localStorage chưa. Đây là vấn đề đặc thù của SSR trong Next.js.

Vấn đề: Khi Next.js render server-side, localStorage không tồn tại. Zustand `persist` phải chờ client mount xong mới có thể đọc storage. Trong khoảng thời gian đó, `accessToken` là `null` — nếu component nào check auth ngay lúc này sẽ thấy user "chưa đăng nhập" và redirect về login nhầm.

`isHydrated` giúp component biết: "Chưa hydrate thì đừng vội render hay redirect, đợi tao tí đã."

```ts
onRehydrateStorage: () => (state) => {
  state?.setHydrated();
},
```

Callback này chạy sau khi Zustand đọc xong localStorage, set `isHydrated: true`. Các component có thể check flag này trước khi render UI liên quan đến auth.

---

## PHẦN 2 — AXIOS INTERCEPTORS & TOKEN REFRESH

---

### Câu 5: JWT-aware Axios interceptors hoạt động như thế nào? Em attach token vào request ra sao?

**Trả lời:**

Em có 2 axios instances:

1. `refreshApi` — instance thuần, **không có interceptor** để tránh vòng lặp vô hạn khi refresh token
2. `api` — instance chính, có đầy đủ interceptors

**Request interceptor** — attach token từ Zustand store vào mỗi request:
```ts
api.interceptors.request.use((config) => {
  const { accessToken } = useAuthStore.getState();
  if (accessToken && config.headers) {
    config.headers.Authorization = `Bearer ${accessToken}`;
  }
  return config;
});
```

Lưu ý: Em gọi `useAuthStore.getState()` thay vì `useAuthStore()` vì interceptor chạy ngoài React component tree, không thể dùng React hook.

---

### Câu 6: Khi JWT hết hạn, em xử lý refresh token như thế nào? Nếu nhiều request cùng 401 thì sao?

**Trả lời:**

Đây là **queue-based refresh pattern** để tránh gọi refresh nhiều lần song song.

```ts
let isRefreshing = false;
let queue: { resolve, reject }[] = [];
```

Flow hoạt động:

**Trường hợp 1 — Request đầu tiên nhận 401:**
- Set `isRefreshing = true`, gọi `refreshApi.post("/auth/refresh-token")`
- Nhận token mới, update Zustand store và cookie
- Giải tỏa queue: gọi `cb.resolve(newToken)` cho tất cả request đang chờ
- Thực hiện lại original request với token mới

**Trường hợp 2 — Request thứ 2, 3... nhận 401 trong khi đang refresh:**
```ts
if (isRefreshing) {
  return new Promise((resolve, reject) => {
    queue.push({
      resolve: (token) => {
        originalRequest.headers["Authorization"] = `Bearer ${token}`;
        resolve(api(originalRequest));
      },
      reject: (err) => reject(err),
    });
  });
}
```
Thay vì gọi refresh lần nữa, chúng "xếp hàng" trong `queue`. Khi refresh xong, tất cả được giải tỏa với token mới.

**Khi refresh thất bại:**
- Logout user khỏi Zustand, xóa cookie
- Redirect về `/auth/login`
- Reject toàn bộ queue

---

### Câu 7: Tại sao lại cần 2 axios instances riêng biệt — `refreshApi` và `api`?

**Trả lời:**

Để tránh vòng lặp vô hạn (infinite loop).

Nếu chỉ có 1 instance `api` và dùng nó để gọi refresh token:
1. Request bình thường → 401
2. Interceptor gọi `api.post("/auth/refresh-token")` để refresh
3. Request refresh này cũng đi qua response interceptor
4. Nếu refresh cũng trả 401, interceptor lại cố refresh tiếp → vòng lặp vô hạn

Giải pháp: `refreshApi` là instance "sạch", không đính interceptor 401-retry. Nó chỉ có nhiệm vụ duy nhất là lấy token mới bằng `httpOnly refresh token cookie` (vì `withCredentials: true`).

Em cũng check điều kiện để không retry với login/refresh endpoint:
```ts
!originalRequest.url?.includes("/auth/login") &&
!originalRequest.url?.includes("/auth/refresh-token")
```

---

### Câu 8: Tại sao lưu `accessToken` trong Zustand store thay vì chỉ đọc từ cookie?

**Trả lời:**

Hai lý do chính:

1. **Accessibility:** Cookie `httpOnly` không đọc được từ JavaScript (đó là security feature của httpOnly). Cookie non-httpOnly thì đọc được nhưng dễ bị XSS. Zustand in-memory nhanh và accessible hơn cho request interceptor.

2. **Reactivity:** Khi token được refresh, em cần update cả Zustand lẫn cookie:
```ts
useAuthStore.getState().setAccessToken(newToken);
Cookies.set("accessToken", newToken, { expires: new Date(decoded.exp * 1000) });
```
Cookie phục vụ Next.js middleware (chạy server-side, đọc được cookie), còn Zustand phục vụ axios interceptor (chạy client-side). Hai nơi cần đồng bộ.

---

## PHẦN 3 — EXAM EDITOR (Part → Group → Question)

---

### Câu 9: Khi em xóa một Part, làm thế nào để `partIndex` của các Part còn lại vẫn đúng?

**Trả lời:**

Trong `exam-editor.tsx`, sau khi filter bỏ Part bị xóa, em map lại để reindex:

```ts
const deletePart = (index: number) => {
  const newParts = value.parts
    .filter((_, i) => i !== index)
    .map((p, i) => ({ ...p, partIndex: i + 1 }));
  onChange({ ...value, parts: newParts });
};
```

`partIndex` được set thành `i + 1` (1-based) để luôn liên tục sau khi xóa. Ví dụ xóa Part 2 trong [1,2,3] → kết quả là [1,2] với partIndex đúng, không phải [1,3].

---

### Câu 10: Khi thêm hoặc xóa một Question, các `questionIndices` trong QuestionGroup có bị sai không? Em xử lý như thế nào?

**Trả lời:**

Đây là một trong những logic phức tạp nhất của editor. Em xử lý theo 2 chiều:

**Khi THÊM câu (insert):**
```ts
const insertedIndex = index + 1; // Vị trí chèn (1-based)
const newGroups = (part.questionGroups || []).map(g => {
  const newIndices = g.questionIndices?.map(idx =>
    idx >= insertedIndex ? idx + 1 : idx
  );
  return { ...g, questionIndices: newIndices || [] };
});
```
Mọi question index ≥ vị trí chèn đều được tăng lên 1. Ví dụ chèn câu vào vị trí 3, các câu từ 3 trở đi thành 4, 5, 6...

**Khi XÓA câu:**
```ts
const newGroups = (part.questionGroups || []).map(g => {
  const newIndices = g.questionIndices
    ?.filter(idx => idx !== deletedQuestionIndex) // Loại bỏ câu bị xóa
    .map(idx => idx > deletedQuestionIndex ? idx - 1 : idx); // Dịch xuống
  return { ...g, questionIndices: newIndices || [] };
});
```
Câu bị xóa được remove khỏi group, các câu có index lớn hơn được giảm xuống 1.

---

### Câu 11: Làm thế nào để ngăn một Question bị assign vào 2 QuestionGroup khác nhau?

**Trả lời:**

Trong `group-editor.tsx`, mỗi `GroupEditor` nhận prop `allOtherUsedIndices` — là tập hợp tất cả questionIndex đã được assign bởi các group *khác*:

```ts
// Trong part-editor.tsx
const allOtherUsedIndices = part.questionGroups
  .filter((_, i) => i !== gIndex)
  .flatMap(g => g.questionIndices || []);
```

Khi user thêm index mới vào group, em check conflict:
```ts
const conflicted = newIndices.filter(n => allOtherUsedIndices.includes(n));
if (conflicted.length > 0) {
  message.error(`Câu ${conflicted.join(", ")} đã nằm trong nhóm khác!`);
  const validIndices = newIndices.filter(n => !conflicted.includes(n));
  onChange({ ...group, questionIndices: [...new Set(validIndices)] });
}
```

Nếu có conflict, hiện error message và chỉ giữ lại những index hợp lệ (không conflict). Em cũng dùng `new Set()` để loại bỏ duplicate trong cùng một group.

---

### Câu 12: Em render các loại câu hỏi khác nhau (single choice, multiple, essay...) như thế nào? Có dùng pattern gì đặc biệt không?

**Trả lời:**

Em dùng **conditional rendering** dựa trên `questionType`, không dùng pattern fancy như component map hay factory. Code trong `question-editor.tsx`:

```ts
// Selector đáp án — single vs multiple
<Select
  mode={isMultiple ? "multiple" : undefined}
  value={isMultiple ? currentAns.correctAnswer : currentAns.correctAnswer?.[0]}
  onChange={(val) => {
    const finalVal = Array.isArray(val) ? val : [val];
    // ...
  }}
/>

// Essay và FILL_IN_THE_BLANK dùng Input text thay vì Select
{isEssay || q.questionType === QuestionType.FILL_IN_THE_BLANK ? (
  <Input ... />
) : (
  <Select ... />
)}

// Options chỉ hiện với non-essay
{q.questionType?.toUpperCase() !== QuestionType.ESSAY && q.options && (
  <div className="grid grid-cols-2 gap-3">
    {q.options.map(...)}
  </div>
)}
```

`QuestionType` là enum constant để tránh magic strings. Khi thêm câu mới, essay không có options còn các loại khác có sẵn 4 options A, B, C, D.

---

### Câu 13: Tại sao `answersState` lại được tách riêng khỏi `ExamData`?

**Trả lời:**

Đây là separation of concerns. `ExamData` (cấu trúc đề thi) và `answersState` (đáp án + điểm) là 2 concern khác nhau vì:

1. **Khác nhau về schema:** `ExamData` theo cấu trúc phân cấp Part → Group → Question. `answersState` là flat map: `Record<questionIndex, { correctAnswer: string[], points: number }>`. Flat map dễ lookup hơn khi cần đọc đáp án của câu số X.

2. **Khác nhau về lifecycle:** Khi AI sinh đề xong, editor nhận `ExamData` nhưng `answersState` ban đầu là rỗng, giáo viên phải điền vào. Tách riêng giúp clear hơn.

3. **Dễ build payload lúc save:** Khi submit, em chỉ cần merge hai state: loop qua `answersState`, tìm question tương ứng trong `ExamData`, ghép lại thành API payload.

---

## PHẦN 4 — MEDIA UPLOAD & CLOUDINARY

---

### Câu 14: Upload flow trong `MediaUploader` hoạt động như thế nào? Tại sao `beforeUpload` trả về `false`?

**Trả lời:**

`MediaUploader` dùng Ant Design `Upload` component với custom upload logic:

```ts
const handleUpload = async (file: File) => {
  try {
    const res = await uploadMutation.mutateAsync(file);
    const data = res.data.data;
    onUploadSuccess({
      url: data.url,
      publicId: data.public_id,
      resourceType: data.resource_type
    });
    message.success("Upload thành công");
  } catch (error) {
    message.error("Upload thất bại");
  }
  return false; // ← Quan trọng
};
```

`return false` trong `beforeUpload` của Ant Design có nghĩa là **ngăn component tự động upload**. Nếu return `true` hoặc không return gì, Ant Design sẽ gửi file lên URL trong prop `action`. Em return `false` để tự handle upload thủ công bằng `uploadMutation.mutateAsync(file)` — gọi thẳng đến backend của mình, không qua Cloudinary trực tiếp từ client.

---

### Câu 15: Khi xóa media, tại sao cần xóa cả trên Cloudinary chứ không chỉ xóa reference?

**Trả lời:**

Nếu chỉ xóa reference (URL trong state), file vẫn tồn tại trên Cloudinary và chiếm storage quota. Cloudinary tính phí theo storage và bandwidth nên cần clean up.

Trong `group-editor.tsx` và `part-editor.tsx`, em gọi mutation xóa trước, sau đó mới update state:

```ts
const handleRemoveMedia = async (mIndex: number, placeholder: MediaPlaceholder) => {
  try {
    if (placeholder.publicId) {
      await deleteMedia.mutateAsync({
        publicId: placeholder.publicId,
        resourceType: placeholder.mediaType === 'audio' ? 'video' : 'image',
      });
    }
    const updated = group.mediaPlaceholders?.filter((_, i) => i !== mIndex);
    onChange({ ...group, mediaPlaceholders: updated || [] });
  } catch (error) {
    message.error("Lỗi xóa file");
  }
};
```

Chú ý: audio trong Cloudinary lưu với `resourceType: 'video'` (Cloudinary gộp audio/video vào cùng một resource type), nên cần map lại: `mediaType === 'audio' ? 'video' : 'image'`.

---

## PHẦN 5 — TANSTACK QUERY

---

### Câu 16: `staleTime: Infinity` trong `useExamDetail` nghĩa là gì? Khi nào dữ liệu được refetch?

**Trả lời:**

`staleTime: Infinity` nghĩa là data được coi là "fresh" mãi mãi — TanStack Query **không bao giờ tự động refetch** dựa trên thời gian.

```ts
export const useExamDetail = (examId: string) => {
  return useQuery({
    queryKey: [...EXAM_QUERY_KEY, "detail", examId],
    queryFn: async () => { ... },
    staleTime: Infinity,
    gcTime: 60 * 60 * 1000, // Cache 1 giờ rồi xóa
  });
};
```

Em dùng `Infinity` vì exam detail (cấu trúc đề thi) là dữ liệu rất ít thay đổi. Một khi giáo viên đã publish đề, nội dung gần như không đổi. Không cần background refetch khi user switch tab hay focus lại window.

Data chỉ được refetch khi:
- User manually trigger (ví dụ pull-to-refresh)
- Em gọi `invalidateQueries` sau khi update
- Cache bị garbage collect sau 1 giờ (`gcTime`)

Ngược lại, `useExamByExamSet` có `staleTime: 5 * 60 * 1000` (5 phút) vì danh sách đề thi thay đổi thường xuyên hơn (thêm/xóa đề).

---

### Câu 17: Sau khi create/update/delete Exam thành công, tại sao gọi `invalidateQueries({ queryKey: EXAM_QUERY_KEY })`?

**Trả lời:**

`EXAM_QUERY_KEY = ["exams"]`. Khi invalidate với key này, TanStack Query mark **tất cả queries có prefix "exams"** là stale, bao gồm:
- `["exams", "exam-set", examSetId, params]` — danh sách đề theo bộ đề
- `["exams", "detail", examId]` — chi tiết đề
- Mọi query nào có `queryKey` bắt đầu bằng `"exams"`

Điều này đảm bảo sau khi tạo đề mới, danh sách trên UI được refetch và hiển thị item mới ngay lập tức, không cần user reload trang.

```ts
export const useCreateExam = () => {
  const qc = useQueryClient();
  return useMutation({
    mutationFn: ExamService.create,
    onSuccess: () => {
      qc.invalidateQueries({ queryKey: EXAM_QUERY_KEY }); // ["exams"]
    },
  });
};
```

TanStack Query dùng **prefix matching**: invalidate `["exams"]` sẽ invalidate mọi query có array bắt đầu bằng `"exams"`.

---

## PHẦN 6 — CÂU HỎI "SÂU" VỀ TRADE-OFFS & LESSONS LEARNED

---

### Câu 18: Em nhận thấy `any` type xuất hiện nhiều ở props. Nếu làm lại, em sẽ type strict hơn như thế nào?

**Trả lời:**

Đúng, trong code em có một số `uploadHook: any`, `answersState: any` — đây là kỹ thuật nợ (tech debt) trong quá trình build nhanh. Nếu làm lại, em sẽ:

1. Define interface rõ ràng cho upload hook:
```ts
interface UploadHook {
  mutateAsync: (file: File) => Promise<{ data: { data: CloudinaryResponse } }>;
  isPending: boolean;
}
```

2. Type `answersState`:
```ts
type AnswersState = Record<number, { correctAnswer: string[]; points: number }>;
```

3. Với `ExamData` và các nested types, em đã có interface trong `@/shares/types/object` nhưng chưa apply đủ chặt ở mọi nơi.

Việc dùng `any` làm mất đi lợi ích của TypeScript — compiler không catch được lỗi kiểu dữ liệu, IDE không suggest autocomplete. Nhìn lại, em sẽ dành thêm 20-30% thời gian để type properly ngay từ đầu.

---

### Câu 19: Error handling trong Exam Editor còn thiếu gì? Em sẽ cải thiện như thế nào?

**Trả lời:**

Nhìn lại code, em thấy một số điểm còn thiếu:

1. **Không có validation trước khi save:** Editor cho phép lưu câu hỏi không có đáp án, điểm = 0. README đề cập `validateBeforeSave` nhưng em chưa implement đầy đủ trong editor component.

2. **Media upload error không rollback state:** Nếu upload thành công nhưng sau đó API lưu đề bị fail, media đã up lên Cloudinary orphaned (không ai xóa).

3. **Optimistic update:** Hiện tại chưa có, mọi thao tác đều chờ server confirm rồi mới update UI. Với editor local state thì không cần, nhưng nếu add save-to-draft feature thì cần.

Cải thiện: Thêm Zod schema để validate `ExamData` trước khi submit, hiện error inline thay vì chỉ dùng `message.error()`.

---

### Câu 20: Nếu nhà tuyển dụng hỏi "Project này khó nhất ở đâu?", em trả lời gì?

**Trả lời (gợi ý):**

Khó nhất là đồng bộ `questionIndices` trong QuestionGroup khi insert/delete câu hỏi. Ban đầu em chỉ xóa câu khỏi mảng `questions` mà quên update `questionGroups`, dẫn đến group vẫn reference index cũ (giờ không còn tồn tại hoặc trỏ sang câu sai).

Để fix, em phải viết logic "cascade update" — khi insert câu tại vị trí X, tất cả group có index ≥ X phải tăng lên 1; khi delete câu X, group phải remove X và giảm mọi index > X xuống 1. Logic này dễ sai nếu không test kỹ các edge case (insert đầu, insert cuối, delete câu đang nằm trong group, delete câu không thuộc group nào).

Đây cũng là lý do em thấy cần unit test nhiều hơn cho các pure transformation functions này — chúng không phụ thuộc UI nên rất dễ test độc lập.

---
## PHẦN 7 — SSE & REALTIME (GIÁM THỊ DASHBOARD)

---

### Câu 21: `useExamSSE` — tại sao dùng `@microsoft/fetch-event-source` thay vì `EventSource` native?

**Trả lời:**

`EventSource` native của browser có hạn chế lớn: **không hỗ trợ custom headers**. Để gửi JWT Authorization header lên server, em cần `@microsoft/fetch-event-source` — library wrap `fetch` để hỗ trợ SSE với đầy đủ headers:

```ts
await fetchEventSource(url, {
  method: 'GET',
  headers: {
    Authorization: `Bearer ${accessToken}`,  // Không thể gửi qua EventSource native
    Accept: 'text/event-stream',
  },
  signal: abortControllerRef.current?.signal,
  async onopen(response) {
    if (response.ok && response.headers.get('content-type')?.includes('text/event-stream')) {
      setIsConnected(true);
    } else {
      throw new Error(`SSE Connection failed: ${response.status}`);
    }
  },
  onmessage(msg) {
    const eventPayload: ExamSessionEvent = JSON.parse(msg.data);
    setLatestEvent(eventPayload);
  },
  onclose() {
    setIsConnected(false);
    throw new Error('Connection closed - force retry'); // Trigger auto-reconnect
  },
  onerror(err) {
    setIsConnected(false);
    // Không throw → fetchEventSource tự retry với exponential backoff
  }
});
```

**Reconnect logic:** Trong `onclose` em throw error để force reconnect ngay. Trong `onerror` em không throw để thư viện tự retry. Sự khác biệt: `onclose` = server chủ động đóng (cần reconnect ngay), `onerror` = lỗi network (để thư viện tự backoff).

**`AbortController`** cleanup khi component unmount:
```ts
return () => {
  abortControllerRef.current?.abort();
  setIsConnected(false);
};
```

---

### Câu 22: SSE nhận những event nào? Giám thị xử lý từng event như thế nào?

**Trả lời:**

```ts
export type ExamSessionEventType =
  | 'STUDENT_JOINED'         // Thí sinh vào phòng thi
  | 'STUDENT_SUBMITTED'      // Thí sinh nộp bài
  | 'STUDENT_VIOLATION'      // Vi phạm detect từ camera/tab switch
  | 'SESSION_STATUS_CHANGED' // Ca thi pause/resume/end
  | 'TIME_WARNING'           // Cảnh báo còn ít thời gian
  | 'RETAKE_GRANTED'         // Cấp quyền thi lại
  | 'ATTEMPT_PAUSED'         // Tạm dừng một thí sinh
  | 'ATTEMPT_RESUMED'        // Tiếp tục một thí sinh
  | 'ATTEMPT_DISCONNECTED'   // Thí sinh mất kết nối
  | 'TEACHER_COLLECTED'      // Giáo viên thu bài
  | 'WARNING' | 'REPRIMAND' | 'SUSPENSION'
```

Monitoring dashboard subscribe `latestEvent` để update student list realtime mà không cần polling:
```ts
const { latestEvent } = useExamSSE(examSessionId);

useEffect(() => {
  if (!latestEvent) return;
  switch (latestEvent.type) {
    case 'STUDENT_VIOLATION':
      queryClient.invalidateQueries({ queryKey: [EXAM_ATTEMPT_KEY, examSessionId] });
      break;
    case 'ATTEMPT_PAUSED':
      // Update status của student đó trong cache
      break;
  }
}, [latestEvent]);
```

Thí sinh phía client cũng nhận event từ SSE — ví dụ `ATTEMPT_PAUSED` → show màn hình "Bài thi tạm dừng", `TEACHER_COLLECTED` → auto redirect về trang kết quả.

---

## PHẦN 8 — MEDIAPIPE & CAMERA (EXAM MONITOR)

---

### Câu 23: MediaPipe FaceMesh được load như thế nào? Tại sao không install qua npm?

**Trả lời:**

```ts
const loadScript = (src: string) => {
  return new Promise((resolve, reject) => {
    if (document.querySelector(`script[src="${src}"]`)) return resolve(true); // Tránh load lại
    const script = document.createElement("script");
    script.src = src;
    script.crossOrigin = "anonymous";
    script.onload = () => resolve(true);
    script.onerror = reject;
    document.head.appendChild(script);
  });
};

// Load song song 2 scripts
await Promise.all([
  loadScript("https://cdn.jsdelivr.net/npm/@mediapipe/face_mesh/face_mesh.js"),
  loadScript("https://cdn.jsdelivr.net/npm/@mediapipe/camera_utils/camera_utils.js")
]);
```

**Tại sao CDN thay vì npm:** MediaPipe WASM bundle rất lớn (~10MB+). Nếu install npm, bundle sẽ được include vào JS chunk của app → tăng đáng kể initial load time cho tất cả trang, kể cả trang không cần proctoring. Load động từ CDN chỉ khi component `ExamMonitor` mount → chỉ tốn bandwidth khi thí sinh vào thi.

Guard `document.querySelector(`script[src="${src}"]`)` tránh load lại script đã có — React Strict Mode mount component 2 lần trong dev.

Sau khi scripts load, khởi tạo và gửi từng frame vào FaceMesh:
```ts
cameraRef.current = new window.Camera(videoRef.current, {
  onFrame: async () => {
    await faceMeshRef.current.send({ image: videoRef.current });
  },
  width: 640, height: 480,
});
```

---

### Câu 24: Logic phát hiện vi phạm — ngưỡng Yaw 0.07 và Pitch 0.06 được tính như thế nào?

**Trả lời:**

```ts
const landmarks = detectedFaces[0];
const nose = landmarks[1];        // Mũi
const leftEye = landmarks[33];    // Mắt trái
const rightEye = landmarks[263];  // Mắt phải
const chin = landmarks[152];      // Cằm
const forehead = landmarks[10];   // Trán

const eyeCenter_x = (leftEye.x + rightEye.x) / 2;
const yaw_diff = nose.x - eyeCenter_x;      // Quay ngang: mũi lệch khỏi center 2 mắt

const faceCenter_y = (forehead.y + chin.y) / 2;
const pitch_diff = nose.y - faceCenter_y;   // Cúi: mũi lệch khỏi center trán-cằm

if (Math.abs(yaw_diff) > 0.07 || pitch_diff > 0.06) {
  violationFound = FraudType.NO_FACE_DETECTED;
}
```

MediaPipe trả về tọa độ **normalized 0-1** theo kích thước frame. Ngưỡng được xác định bằng empirical testing:

- **`|yaw_diff| > 0.07`**: Mũi lệch > 7% chiều rộng so với center 2 mắt → quay đầu nhìn tài liệu
- **`pitch_diff > 0.06`**: Mũi thấp hơn center trán-cằm > 6% → cúi xuống nhìn bàn

Chỉ check `pitch_diff > 0.06` (dương) vì ngẩng đầu lên (âm) ít nghiêm trọng hơn cúi xuống nhìn tài liệu.

3 case được check theo thứ tự ưu tiên: `detectedFaces.length === 0` → `> 1` → kiểm tra pose. Nếu phát hiện nhiều mặt, không cần check pose của mặt đầu tiên.

---

### Câu 25: Delay 1 giây trước khi báo vi phạm — implement và lý do?

**Trả lời:**

```ts
const violationTimerRef = useRef<NodeJS.Timeout | null>(null);
const currentViolationTypeRef = useRef<FraudType | null>(null);

if (violationFound) {
  if (currentViolationTypeRef.current !== violationFound) {
    clearViolationTimer();
    currentViolationTypeRef.current = violationFound;

    violationTimerRef.current = setTimeout(() => {
      const evidence = captureEvidence();
      onViolation(violationFound!, evidence);
      message.warning(msg);
      clearViolationTimer();
    }, 1000);
  }
} else {
  clearViolationTimer(); // Tư thế đúng → reset timer
}
```

1 giây để lọc **transient false positive**: thí sinh quay đầu nhìn đồng hồ 0.3s rồi quay lại, camera glitch một frame, ánh sáng thay đổi đột ngột mất track 0.5s — đây không phải gian lận thật.

**`currentViolationTypeRef`** theo dõi violation đang đếm: nếu vi phạm thay đổi loại (NO_FACE → MULTIPLE_FACES), timer reset từ đầu — mỗi loại vi phạm phải sustain đủ 1 giây mới tính.

Dùng **`ref` thay vì `state`** cho cả hai vì không cần re-render — chỉ là runtime tracking không ảnh hưởng UI. `violationTimerRef` cũng không phải state vì `clearTimeout` là side effect, không phải update UI.

---

### Câu 26: `captureEvidence` — tại sao cần mirror flip khi chụp ảnh bằng chứng?

**Trả lời:**

```ts
const captureEvidence = (): string => {
  const canvas = canvasRef.current;
  const video = videoRef.current;
  canvas.width = video.videoWidth;
  canvas.height = video.videoHeight;
  const ctx = canvas.getContext('2d');
  if (ctx) {
    ctx.translate(canvas.width, 0);
    ctx.scale(-1, 1);          // Flip ngang — un-mirror
    ctx.drawImage(video, 0, 0, canvas.width, canvas.height);
    return canvas.toDataURL('image/jpeg', 0.6);
  }
  return '';
};
```

Video hiển thị với `style={{ transform: 'scaleX(-1)' }}` — mirror để thí sinh thấy như gương (tự nhiên). Nhưng ảnh gửi lên server cần **ảnh thực** (không mirror) để giám thị nhận diện đúng trái/phải.

`captureEvidence` un-mirror bằng canvas transform trước khi draw → ảnh được restore về chiều thực tế.

Quality 60% (`toDataURL('image/jpeg', 0.6)`) — đủ rõ để nhận diện người nhưng file size nhỏ. Canvas hidden (`className="hidden"`) — chỉ dùng để capture, không render ra UI.

---

## PHẦN 9 — AUTO-SAVE, TIMER & SECURITY

---

### Câu 27: `useExamAutoSave` — delta-based save và stale closure problem?

**Trả lời:**

```ts
const lastSavedAnswersRef = useRef<string>("");
const currentUserAnswersRef = useRef(userAnswers);

useEffect(() => {
  currentUserAnswersRef.current = userAnswers; // Sync ref mỗi render
}, [userAnswers]);

useEffect(() => {
  if (!examAttemptId) return;
  const timer = setInterval(() => {
    const currentAnswers = currentUserAnswersRef.current;
    const answersString = JSON.stringify(currentAnswers);

    if (currentAnswers.length > 0 && answersString !== lastSavedAnswersRef.current) {
      saveDraft({ examSessionId, changes: currentAnswers }, {
        onSuccess: () => {
          lastSavedAnswersRef.current = answersString; // Cập nhật checkpoint
        }
      });
    }
  }, 10000);
  return () => clearInterval(timer);
}, [examAttemptId, examSessionId, saveDraft]);
```

**Stale closure:** `setInterval` callback capture `userAnswers` tại thời điểm `useEffect` chạy. Nếu thí sinh trả lời thêm sau đó, closure vẫn giữ giá trị cũ → không save được câu mới. `currentUserAnswersRef` giải quyết: mỗi render sync ref → interval đọc `.current` luôn có giá trị mới nhất.

**`lastSavedAnswersRef` làm checkpoint:** So sánh `JSON.stringify` của answers hiện tại vs lần save thành công cuối. Nếu giống → skip API call. Tiết kiệm bandwidth khi thí sinh không thay đổi gì trong 10 giây (ví dụ đang đọc câu hỏi dài).

---

### Câu 28: `useExamTimer` — countdown survive reload ra sao? Ngưỡng 5 giây làm gì?

**Trả lời:**

```ts
const [endTime, setEndTime] = useState<number>(() => {
  // Lazy initializer: đọc localStorage ngay khi component mount
  const saved = localStorage.getItem(`exam_endtime_${examSessionId}`);
  return saved ? parseInt(saved, 10) : Date.now();
});

useEffect(() => {
  // Ưu tiên tính từ server timestamp
  if (lastResumedAt) {
    const lastResumedTimeMs = new Date(lastResumedAt).getTime();
    const remainingMs = (duration * 60 * 1000) - (consumedTime * 1000);
    calculatedEndTime = lastResumedTimeMs + remainingMs;
  }

  if (saved) {
    const diff = Math.abs(savedTime - calculatedEndTime);
    if (diff > 5000 && lastResumedAt) {
      // Lệch > 5s VÀ có server timestamp mới → dùng server time
      localStorage.setItem(storageKey, calculatedEndTime.toString());
      setEndTime(calculatedEndTime);
    } else {
      setEndTime(savedTime); // Tin localStorage
    }
  }
}, [examSessionId, duration, consumedTime, lastResumedAt]);
```

**Survive reload:** `useState` lazy initializer (`() => { ... }`) đọc localStorage trước render đầu tiên — không cần thêm useEffect. Countdown tính `timeLeft = endTime - Date.now()` mỗi tick.

**Ngưỡng 5s:** Server trả về `consumedTime` có thể có sai số nhỏ (network latency, server rounding). Không nên update `endTime` mỗi lần `consumedTime` thay đổi nhỏ → gây flicker. Chỉ sync lại khi lệch > 5s — đủ để detect giám thị pause/resume thực sự.

**`lastResumedAt`** là server timestamp khi resume ca thi. Dùng server time tránh **clock skew** giữa máy thí sinh và server.

---

### Câu 29: `useExamSecurity` — heartbeat 5 giây nhưng Redis TTL 15 giây. Tại sao margin 3x?

**Trả lời:**

```ts
sendHeartbeat(examAttemptId); // Ngay lập tức khi mount

const interval = setInterval(() => {
  sendHeartbeat(examAttemptId);
}, 5000); // Mỗi 5 giây
```

**3x margin:** Với Redis TTL = 15s và heartbeat mỗi 5s, phải mất **3 heartbeat liên tiếp thất bại** mới trigger "offline" ở server. Một heartbeat bị drop do network blip nhất thời sẽ không ảnh hưởng — thí sinh không bị đánh dấu disconnect nhầm.

Ngoài interval còn gửi heartbeat ở event đặc biệt:
```ts
window.addEventListener('offline', handleOffline);
document.addEventListener('visibilitychange', handleVisibility); // Tab ẩn
window.addEventListener('beforeunload', handleBeforeUnload);     // Đóng tab
```

`beforeunload` gửi heartbeat cuối trước khi tab đóng — server biết là disconnect chủ động, không phải crash.

---

### Câu 30: Phân biệt 3 loại fraud detection: `visibilitychange` vs `blur` vs `fullscreenchange`?

**Trả lời:**

```ts
// 1. Tab switching: chuyển sang tab khác trong cùng browser
document.addEventListener('visibilitychange', () => {
  if (document.visibilityState === 'hidden')
    reportFraud(FraudType.TAB_SWITCHING);
});

// 2. Window blur: click ra app khác, Alt+Tab
window.addEventListener('blur', () => {
  reportFraud(FraudType.WINDOW_BLUR);
});

// 3. Thoát fullscreen: Escape hoặc F11 (trong useExamFullscreen)
document.addEventListener('fullscreenchange', () => {
  if (!document.fullscreenElement)
    onViolation(FraudType.WINDOW_BLUR);
});
```

**Phân biệt:**
- `visibilitychange → hidden`: Tab được chuyển sang background, browser vẫn foreground. User đang ở cùng browser nhưng xem tab khác.
- `window.blur`: Toàn bộ browser mất focus — user click vào app khác (Word, Calculator) hoặc Alt+Tab ra desktop.
- `fullscreenchange`: Thoát fullscreen bằng Escape/F11 — có thể che giấu browser để nhìn tài liệu bên dưới.

3 loại bổ sung nhau: một số hành vi chỉ trigger một loại. Ví dụ mở DevTools chỉ trigger `blur` chứ không trigger `visibilitychange`.

---

### Câu 31: `useExamSecurity` auto-capture ảnh mặt mỗi 3 phút — dùng `document.querySelector('video')` thay vì ref, tại sao?

**Trả lời:**

```ts
const handleAutoCapture = () => {
  const video = document.querySelector('video'); // Query DOM trực tiếp
  if (!video || video.readyState !== 4) return;  // readyState 4 = HAVE_ENOUGH_DATA

  const canvas = document.createElement('canvas');
  canvas.width = 320; canvas.height = 240;
  const ctx = canvas.getContext('2d');
  ctx.translate(canvas.width, 0);
  ctx.scale(-1, 1);
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height);

  canvas.toBlob((blob) => {
    const faceFile = new File([blob], `verify_${examAttemptId}_${Date.now()}.jpg`, { type: "image/jpeg" });
    verifyFace({ examAttemptId, face: faceFile, occurredAt: new Date() });
  }, "image/jpeg", 0.7);
};

const firstShot = setTimeout(handleAutoCapture, 60000); // Lần đầu sau 1 phút
const interval = setInterval(handleAutoCapture, VERIFY_INTERVAL); // Mỗi 3 phút
```

`useExamSecurity` là custom hook — không render UI, không có ref đến video element. `ExamMonitor` component (khác file) mới có video element. Thay vì truyền ref qua props, em query DOM trực tiếp — đơn giản hơn và không cần refactor.

`video.readyState !== 4` guard tránh capture khi camera chưa ready (tránh ảnh đen hoặc frame đầu chưa load).

`setTimeout(60s)` cho lần đầu — thí sinh có thời gian setup camera trước khi bị capture định kỳ.

---

### Câu 32: `useExamSubmit` — `isSubmittedRef` guard tại sao dùng ref thay vì state?

**Trả lời:**

```ts
const isSubmittedRef = useRef(false);

const submitExamAction = () => {
  if (isSubmitting) return;           // Guard 1: mutation đang pending
  isSubmittedRef.current = true;      // Synchronous, immediate
  setIsCameraActive(false);
  localStorage.removeItem(`exam_endtime_${examSessionId}`);
  localStorage.removeItem(`exam_progress_${examSessionId}`);

  submitExam(payload, {
    onError: (error) => {
      if (error.response?.status !== 401) {
        setTimeout(() => { isSubmittedRef.current = false; }, 2000); // Cho retry sau 2s
      }
    }
  });
};

const handleSubmit = (isAutoSubmit = false) => {
  if (isSubmittedRef.current || isSubmitting) return; // Guard 2: đã submit rồi
  // ...
};
```

**Tại sao ref:** `isSubmittedRef.current = true` là **synchronous** — có hiệu lực ngay tức thì. Nếu dùng `useState`, React batch state updates → set state không apply ngay → trong khoảng trống giữa set và re-render, user có thể double-click submit → 2 request được gửi.

**Auto-submit khi hết giờ:**
```ts
if (isAutoSubmit) {
  isSubmittedRef.current = true;
  message.warning({ content: "Đã hết giờ! Đang tự động nộp bài.", duration: 0 });
  submitExamAction();
  return;
}
```
`duration: 0` giữ message không tự đóng — user không dismiss được trong khi đang submit.

---

## PHẦN 10 — STUDENT QUESTION INTERFACE

---

### Câu 33: `StudentQuestion` render 5 loại câu hỏi. Tại sao SINGLE_CHOICE và TRUE_FALSE fallthrough cùng case?

**Trả lời:**

```ts
switch (type) {
  case QuestionType.SINGLE_CHOICE:
  case QuestionType.TRUE_FALSE:     // Fallthrough — cùng dùng Radio.Group
    return (
      <Radio.Group onChange={(e) => onChange(e.target.value)} value={value}>
        {options?.map((opt) => (
          <Radio key={opt.label} value={opt.label}
            className={value === opt.label ? activeClass : inactiveClass}>
            <span className="font-bold">{opt.label}.</span> {opt.text}
          </Radio>
        ))}
      </Radio.Group>
    );

  case QuestionType.MULTIPLE_CHOICE:
    return <Checkbox.Group onChange={(vals) => onChange(vals)} value={value || []} />;

  case QuestionType.FILL_IN_THE_BLANK:
    return <Input value={value} onChange={(e) => onChange(e.target.value)} />;

  case QuestionType.ESSAY:
    return (
      <div>
        <Input.TextArea value={value?.content} ... />
        <Upload beforeUpload={handleUpload}>...</Upload>
      </div>
    );
}
```

TRUE_FALSE thực chất là SINGLE_CHOICE với chỉ 2 options (Đúng/Sai). Backend xử lý chấm điểm khác nhau nhưng UI giống hệt nhau → fallthrough hợp lý, không cần duplicate code.

Essay có data structure phức tạp hơn: `{ content: string, file_metadata: FileItem[] }` thay vì chỉ `string` — vì essay cho phép đính kèm file PDF.

**`activeClass`/`inactiveClass`** tạo visual feedback rõ ràng: option được chọn nổi bật với màu primary và border, giúp thí sinh confirm lựa chọn của mình.

---

### Câu 34: Essay file upload — `handleUpload` trả về `false`. Server trả về gì?

**Trả lời:**

```ts
const handleUpload = async (file: File) => {
  try {
    const response = await uploadFile(file);
    const { objectKey, signedUrl } = response.data.data;

    onChange({
      content: value?.content || "",
      file_metadata: [...(value?.file_metadata || []), { signedUrl, objectKey }]
    });
  } catch (error) {
    message.error("Lỗi khi tải file");
  }
  return false; // Ngăn Ant Design Upload tự upload lên action URL
};
```

Server trả về 2 fields:
- `signedUrl`: URL tạm thời để download/preview file (link trực tiếp đến S3)
- `objectKey`: Path trong S3 để delete sau

`return false` trong `beforeUpload` ngăn Ant Design Upload tự xử lý upload (nó sẽ gửi lên prop `action` nếu không block). Em tự handle upload qua `uploadFile` mutation.

```ts
const handleDelete = async (fileItem: any) => {
  await deleteFile({ filePath: fileItem.objectKey }); // Xóa trên S3 theo objectKey
  onChange({
    ...value,
    file_metadata: value.file_metadata.filter(m => m.objectKey !== fileItem.objectKey)
  });
};
```

Xóa theo `objectKey` (permanent identifier) thay vì `signedUrl` (có thể expire).

---

### Câu 35: `ProgressButton` — tại sao `isAnswered` check phức tạp với nhiều kiểu answer?

**Trả lời:**

```ts
const isAnswered = userAnswer && (() => {
  const ans = userAnswer.answer;
  if (Array.isArray(ans)) return ans.length > 0;        // Multiple choice: ["A", "C"]
  if (typeof ans === 'object' && ans !== null) {
    const hasContent = !!ans.content?.trim();             // Essay text
    const hasFiles = Array.isArray(ans.file_metadata)
      && ans.file_metadata.length > 0;                   // Essay files
    return hasContent || hasFiles;
  }
  return !!ans;                                           // String: "A", "Đáp án"
})();
```

Answer structure khác nhau theo loại câu hỏi:
- Single/True-False/Fill-in: `answer = "A"` → `!!ans`
- Multiple choice: `answer = ["A", "C"]` → `ans.length > 0`
- Essay: `answer = { content: "...", file_metadata: [...] }` → cần check cả text lẫn file

Nếu chỉ check `!!ans`, essay object `{ content: "", file_metadata: [] }` vẫn truthy dù chưa thực sự trả lời. Check phức tạp hơn đảm bảo button chỉ highlight khi có nội dung thực sự.

Button scroll đến câu hỏi:
```ts
document.getElementById(`question-${question.questionId}`)
  ?.scrollIntoView({ behavior: 'smooth', block: 'center' });
```

---

## PHẦN 11 — MONITORING DASHBOARD (GIÁM THỊ)

---

### Câu 36: `MonitoringTableTab` — per-row loading state cho action buttons dùng `variables` từ TanStack Query?

**Trả lời:**

```ts
const { mutate: pauseAttempt, isPending: isPausing, variables: pauseVars } = usePauseStudentAttempt();

// Trong render từng dòng:
<Button
  loading={isPausing && (pauseVars as any)?.studentId === record.studentId}
  icon={<PauseCircleOutlined />}
/>
```

`isPausing` là global flag — true khi bất kỳ pauseAttempt nào đang pending. `pauseVars` là payload của lần mutate đang chạy. So sánh `pauseVars?.studentId === record.studentId` → chỉ button của đúng student đang được xử lý mới hiện loading spinner.

Pattern này tránh phải dùng `useState<string | null>` riêng để track `loadingStudentId` — TanStack Query đã expose `variables` sẵn, tận dụng luôn.

Tương tự với `forceSubmit` (check `forceVars?.data?.studentIds?.includes(record.studentId)`), `saveAnswers` (check `saveVars?.data?.studentId`).

---

### Câu 37: `FraudLogTab` dùng `useDebounce` cho search. Tại sao cần và `setCurrentPage(1)` đặt ở đâu?

**Trả lời:**

```ts
const [searchText, setSearchText] = useState("");
const debouncedSearch = useDebounce(searchText, 500);

const { data } = useFraudDetailList({
  examSessionId,
  studentCode: debouncedSearch || undefined // API chỉ nhận khi có giá trị
});

const handleSearch = (e) => {
  setSearchText(e.target.value);
  setCurrentPage(1); // Reset page ngay khi gõ
};
```

Không có debounce, gõ MSSV "2021601234" (10 ký tự) = 10 API calls. Với debounce 500ms: chỉ gọi sau khi dừng gõ 500ms → ~1 call.

**`setCurrentPage(1)` trong `handleSearch`** (không phải trong useEffect của `debouncedSearch`): page reset **ngay lập tức** khi user bắt đầu gõ, không phải sau 500ms. Nếu đặt trong useEffect của debounced value, user thấy page vẫn giữ nguyên trong 500ms rồi mới nhảy về 1 — UX không mượt.

---

### Câu 38: `StudentListTab` — `isAllAccessGranted` dùng `useMemo` và bulk access toggle hoạt động ra sao?

**Trả lời:**

```ts
const isAllAccessGranted = useMemo(() => {
  if (!data?.data || data.data.length === 0) return false;
  return data.data.every((student) => student.isAccessGranted);
}, [data]);
```

Switch "Quyền truy cập toàn ca thi" là derived state: `true` khi **tất cả** thí sinh trong page đều `isAccessGranted = true`. Một thí sinh nào `false` → switch tắt.

```ts
const handleToggleAllAccess = (checked: boolean) => {
  updateBulkAccess.mutate(
    { examSessionId, data: { isAccessGranted: checked } },
    { onSuccess: () => toast.success(`Đã ${checked ? 'mở' : 'khóa'} quyền truy cập`) }
  );
};
```

Server update toàn bộ registrations → TanStack Query invalidate → list re-fetch → `isAllAccessGranted` recompute.

`useMemo` với `[data]` dependency: loop `every()` chỉ chạy lại khi `data` thay đổi (sau refetch), không chạy lại khi `params`, `isPasswordModalOpen`... thay đổi — tránh O(n) scan không cần thiết.

---

*Tổng cộng: 38 câu hỏi & trả lời cho iTEST (100% từ code thực tế)*
*File nguồn: useExamStore.ts, useAuthStore.ts, axios.ts, exam-editor.tsx, part-editor.tsx, group-editor.tsx, question-editor.tsx, media-render.tsx, media-uploader.tsx, useExamQuery.ts, useExamSSE.ts, exam-monitor.tsx, useExamAutoSave.ts, useExamTimer.ts, useExamSecurity.ts, useExamFullscreen.ts, useExamNetwork.ts, useExamSubmit.ts, student-question.tsx, renderProgressButton.tsx, monitoring-tab.tsx, monitoring-table-tab.tsx, monitoring-filter.tsx, student-card.tsx, student-list-tab.tsx, fraud-log-tab.tsx, examSessionHandling-create-modal.tsx*
