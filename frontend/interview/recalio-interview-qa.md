# Câu hỏi & Trả lời phỏng vấn — Recalio
> Dựa trực tiếp vào code thực tế: `image-occlusion-editor.tsx`, `image-occlusion-card-view.tsx`, `cloze-editor.tsx`, `card-preview.tsx` (×2), `word-item.tsx`, `ai-generate-from-text-tab.tsx`, `ai-generate-from-topic-tab.tsx`, `ai-generate-from-image-tab.tsx`, `manual-tab.tsx`, `csv-import-tab.tsx`, `page.tsx` (create-notes), `page.tsx` (study-session-detail), `review-card.tsx`

---

## PHẦN 1 — IMAGE OCCLUSION EDITOR

---

### Câu 1: SVG-based Image Occlusion editor — user vẽ mask region như thế nào? Em dùng event nào để handle drag?

**Trả lời:**

Em dùng 3 mouse events trên một `div` wrapper bọc quanh ảnh:

```tsx
<div
  ref={imageRef}
  className="cursor-crosshair select-none"
  onMouseDown={handleMouseDown}
  onMouseUp={handleMouseUp}
  onMouseLeave={() => setDrawing(false)}
>
```

**Flow vẽ mask:**

```ts
// MouseDown — ghi nhận điểm bắt đầu
const handleMouseDown = (e: React.MouseEvent) => {
  if (!imageUrl) return
  setDrawing(true)
  setStartPos(getRelativePos(e))
}

// MouseUp — tính toán và lưu mask
const handleMouseUp = (e: React.MouseEvent) => {
  if (!drawing || !startPos) return
  setDrawing(false)
  const end = getRelativePos(e)

  const x = Math.min(startPos.x, end.x)       // Hỗ trợ kéo từ phải sang trái
  const y = Math.min(startPos.y, end.y)
  const width = Math.abs(end.x - startPos.x)
  const height = Math.abs(end.y - startPos.y)

  if (width < 1 || height < 1) return          // Bỏ qua click nhầm

  const newMask: Mask = { x, y, width, height, groupIndex: currentGroup }
  onChange({ imageUrl, masks: [...masks, newMask] })
  setCurrentGroup((g) => g + 1)
}
```

`onMouseLeave` reset `drawing = false` khi chuột ra khỏi vùng ảnh — tránh mask bị "treo" nếu user nhả chuột bên ngoài element.

---

### Câu 2: Tọa độ mask được lưu như thế nào? Tại sao dùng % thay vì pixel?

**Trả lời:**

```ts
const getRelativePos = useCallback((e: React.MouseEvent) => {
  const rect = imageRef.current?.getBoundingClientRect()
  if (!rect) return { x: 0, y: 0 }
  return {
    x: ((e.clientX - rect.left) / rect.width) * 100,
    y: ((e.clientY - rect.top) / rect.height) * 100,
  }
}, [])
```

Tọa độ được convert sang **phần trăm (0-100)** ngay tại thời điểm ghi nhận. Lý do:

**Responsive:** Ảnh render với `w-full` — kích thước thay đổi theo viewport. Nếu lưu pixel (ví dụ x=120px), khi ảnh thu nhỏ xuống 50%, mask sẽ hiển thị sai vị trí. Với %, mask luôn đúng vị trí bất kể ảnh to nhỏ thế nào.

**Consistent với SVG viewBox:** `ImageOcclusionCardView` render mask bằng SVG với `viewBox="0 0 100 100"` và `preserveAspectRatio="none"`. SVG coordinate space từ 0-100 map trực tiếp vào % — không cần convert khi render.

```tsx
// image-occlusion-card-view.tsx
<svg viewBox="0 0 100 100" preserveAspectRatio="none">
  {masks.map((m, i) => (
    <rect x={m.x} y={m.y} width={m.width} height={m.height} ... />
  ))}
</svg>
```

`m.x = 30` → SVG rect bắt đầu ở 30% chiều ngang → CSS absolute position cũng `left: 30%` → hoàn toàn nhất quán.

---

### Câu 3: Grouping mask với Shift key — logic hoạt động như thế nào?

**Trả lời:**

```ts
const [currentGroup, setCurrentGroup] = useState(1)
const [shiftHeld, setShiftHeld] = useState(false)

// Trong handleMouseUp:
const newMask: Mask = {
  x, y, width, height,
  groupIndex: shiftHeld && masks.length > 0
    ? currentGroup - 1   // Shift: dùng lại group index trước đó
    : currentGroup       // Không Shift: group index mới
}
onChange({ imageUrl, masks: [...masks, newMask] })
if (!shiftHeld) setCurrentGroup((g) => g + 1) // Chỉ tăng khi không Shift
```

**Ví dụ:** Ảnh bản đồ có 2 vùng cần che cùng một câu hỏi (ví dụ: hai tỉnh liền kề thuộc cùng một vùng kinh tế):
- Vẽ mask 1 → `groupIndex: 1`, currentGroup tăng lên 2
- Giữ Shift + vẽ mask 2 → `groupIndex: currentGroup - 1 = 1` — cùng group với mask 1, currentGroup không tăng
- Khi ôn tập, cả hai mask sẽ được reveal/hide cùng lúc vì cùng `groupIndex`

`shiftHeld` state được update qua `keydown`/`keyup` event listeners trên window để theo dõi trạng thái phím Shift toàn cục.

---

### Câu 4: `ImageOcclusionCardView` — logic render mask khác nhau ở mặt trước/sau như thế nào?

**Trả lời:**

```tsx
// image-occlusion-card-view.tsx
{masks.map((m, i) => {
  const isTarget = m.groupIndex === variantIndex  // Mask đang được hỏi

  if (showBack && isTarget) return null           // Mặt sau: ẩn mask mục tiêu (reveal)

  const fill = isTarget
    ? "rgba(0,0,0,0.82)"    // Mặt trước: che đen mask mục tiêu
    : showBack
      ? "none"              // Mặt sau: các mask khác trong suốt
      : "rgba(0,0,0,0.35)"  // Mặt trước: các mask khác mờ hơn

  const stroke = isTarget ? "none"
    : showBack ? "rgba(34,197,94,0.5)"   // Mặt sau: viền xanh cho mask đã đúng
    : "rgba(255,255,255,0.4)"

  return <rect x={m.x} y={m.y} ... fill={fill} stroke={stroke} />
})}
```

**Mặt trước (showBack=false):**
- Mask mục tiêu (`isTarget`): fill đen đậm 82% → user không thấy vùng đó
- Mask khác: fill đen mờ 35% → user biết có vùng che nhưng không phải câu hỏi hiện tại

**Mặt sau (showBack=true):**
- Mask mục tiêu: `return null` → biến mất hoàn toàn → reveal ảnh bên dưới
- Mask khác: `fill="none"`, viền xanh dashed → đánh dấu đã được ôn

Khi reveal, nếu mask có `label`, component hiển thị đáp án bên dưới ảnh:
```tsx
{showBack && (() => {
  const revealed = masks.filter((m) => m.groupIndex === variantIndex)
  const label = revealed[0]?.label
  if (!label) return null
  return <div className="bg-emerald-50 ..."><span>{label}</span></div>
})()}
```

---

## PHẦN 2 — CLOZE EDITOR

---

### Câu 5: `ClozeEditor` — user tạo cloze deletion như thế nào? `insertCloze` hoạt động ra sao?

**Trả lời:**

```ts
const insertCloze = useCallback(() => {
  const ta = textareaRef.current
  if (!ta) return
  const start = ta.selectionStart    // Vị trí đầu vùng bôi đen
  const end = ta.selectionEnd        // Vị trí cuối vùng bôi đen
  if (start === end) return           // Không có gì được chọn → bỏ qua

  const selected = text.substring(start, end)
  const cloze = `{{c${currentIndex}::${selected}}}`  // Wrap thành cloze syntax

  // Thay thế vùng bôi đen bằng cloze tag
  const newText = text.substring(0, start) + cloze + text.substring(end)
  onChange({ Text: newText, Extra: extra })
  setCurrentIndex((i) => i + 1)  // Tăng index cho lần tiếp theo

  // Restore cursor sau cloze tag vừa insert
  requestAnimationFrame(() => {
    ta.focus()
    ta.setSelectionRange(start + cloze.length, start + cloze.length)
  })
}, [text, extra, currentIndex, onChange])
```

**Ví dụ:** Text: `"Paris is the capital of France."` User bôi đen `"Paris"` → bấm nút → kết quả: `"{{c1::Paris}} is the capital of France."`. Bấm lần 2 bôi `"France"` → `"{{c1::Paris}} is the capital of {{c2::France}}."`

`requestAnimationFrame` đảm bảo cursor được set sau khi React re-render xong (do `onChange` trigger state update). Nếu set cursor trước re-render, DOM chưa được cập nhật → cursor bị reset về đầu.

---

### Câu 6: Tại sao nút "Cloze" bị `disabled` khi `!textareaRef.current?.selectionEnd`?

**Trả lời:**

```tsx
<Button
  onClick={insertCloze}
  disabled={!textareaRef.current?.selectionEnd}
>
  Cloze (c{currentIndex})
</Button>
```

`selectionEnd` là 0 (falsy) khi textarea không có focus hoặc không có text nào được chọn. Nếu không disabled, user có thể bấm nút khi không có gì được bôi đen → `start === end` → hàm return sớm nhưng vẫn gây confusing UX.

Tuy nhiên đây là một điểm còn hạn chế: `selectionEnd` là thuộc tính của DOM, không phải React state — React không tự re-render khi selection thay đổi. Nên button thực ra chỉ disable khi `selectionEnd = 0` lúc component render, không phản ứng realtime theo selection. Cải thiện tốt hơn là dùng `onSelect` event của textarea để track selection vào state.

---

## PHẦN 3 — CARD PREVIEW & TEMPLATE RENDERING

---

### Câu 7: `CardPreview` (trong create-notes) render template như thế nào? `substitute` function làm gì?

**Trả lời:**

```ts
function substitute(html: string, fieldMap: Record<string, string>, side: "front" | "back"): string {
  let result = html

  // 1. Simple field substitution: {{Word}} → "Hello"
  for (const [key, val] of Object.entries(fieldMap)) {
    result = result.replaceAll(`{{${key}}}`, val)
  }

  // 2. Cloze rendering — khác nhau giữa front và back
  result = result.replace(/{{cloze:([^}]+)}}/g, (_, field) => {
    const value = fieldMap[field.trim()] || ""
    const processed = value.replace(/\{\{c\d+::(.*?)\}\}/g, (_m, content) =>
      side === "back"
        ? `<span class="cloze-reveal font-bold text-terracotta">${content}</span>` // Back: hiện đáp án
        : `<span class="cloze">[...]</span>`                                        // Front: ẩn thành [...]
    )
    return processed || value
  })

  // 3. Type answer field → disabled input (preview mode)
  result = result.replace(/{{type:([^}]+)}}/g, () =>
    `<input type="text" placeholder="..." disabled />`
  )

  // 4. Answer divider → styled HR
  result = result.replace(/<hr id="answer"\s*\/?>/g, '<hr class="my-3 border-beige" />')

  return result
}
```

`fieldMap` được build từ data của word:
```ts
function buildFieldMap(data: CardPreviewData): Record<string, string> {
  return {
    Word: data.word,
    Meaning: data.meaning || "",
    IPA: data.ipa || "",
    Image: data.imageUrl ? `<img src="${data.imageUrl}" ... />` : "",
    Audio: data.audioUrl ? `<button onclick="new Audio('${data.audioUrl}').play()">🔊 Nghe</button>` : "",
    ...data.fields,
  }
}
```

Template HTML từ server (`frontHtml`, `backHtml`) chứa placeholders như `{{Word}}`, `{{Meaning}}`. `substitute` replace chúng bằng giá trị thực, xử lý special syntax như `{{cloze:Text}}`, `{{type:Word}}`.

---

### Câu 8: `CardPreview` trong trang study session detail có flip animation. Nó được implement như thế nào?

**Trả lời:**

Từ `card-preview.tsx` (study session):

```tsx
const [showBack, setShowBack] = useState(false)

<div
  onClick={handleFlip}  // Toggle showBack
  className="[perspective:1000px]"  // Tạo depth 3D
>
  <div className={`
    transition-transform duration-500
    [transform-style:preserve-3d]
    ${showBack ? "[transform:rotateY(180deg)]" : ""}
  `}>
    {/* Mặt trước */}
    <div className="absolute inset-0 [backface-visibility:hidden]">
      <div dangerouslySetInnerHTML={{ __html: card?.frontHtml ?? "" }} />
    </div>

    {/* Mặt sau — đã xoay 180° từ đầu */}
    <div className="absolute inset-0 [backface-visibility:hidden] [transform:rotateY(180deg)]">
      <div dangerouslySetInnerHTML={{ __html: backHtml }} />
    </div>
  </div>
</div>
```

CSS 3D trick:
- `perspective:1000px` trên container tạo không gian 3D
- `transform-style:preserve-3d` cho phép children render trong không gian 3D
- `backface-visibility:hidden` ẩn mặt sau khi đang nhìn mặt trước (và ngược lại)
- Mặt sau mặc định `rotateY(180deg)` → khi card flip thêm 180deg → tổng = 360deg → hiển thị đúng chiều

`backHtml` được xử lý qua `processBackHtml` thay vì `substitute` vì trong study context chỉ cần render mặt sau, không cần xử lý cloze front/back distinction.

---

### Câu 9: Image Occlusion card trong study context — xử lý khác gì so với card thường?

**Trả lời:**

```tsx
const isOcclusion = !!card?.occlusion

if (isOcclusion) {
  return (
    <div onClick={handleFlip} className="cursor-pointer ...">
      <ImageOcclusionCardView
        imageUrl={card.occlusion.imageUrl}
        masks={card.occlusion.masks}
        variantIndex={card.variantIndex}  // Index của mask đang được hỏi
        showBack={showBack}
        compact={compact}
      />
      {card.css && <style>{card.css}</style>}
    </div>
  )
}
```

Occlusion card không dùng flip animation 3D vì nó không có mặt trước/sau theo nghĩa thông thường — thay vào đó, cùng một ảnh được render với mask ở 2 trạng thái khác nhau (che/hiện). `showBack` toggle giữa 2 trạng thái này trong `ImageOcclusionCardView`.

`variantIndex` là key quan trọng — mỗi mask group tạo ra một "variant" riêng. Một note image occlusion có 5 mask groups → 5 cards riêng biệt trong hàng đợi ôn tập, mỗi card có `variantIndex` khác nhau (0, 1, 2, 3, 4).

---

## PHẦN 4 — AI VOCABULARY CREATION FLOW

---

### Câu 10: AI generate flow có 3 input modes (text, topic, image). Code chung ở đâu? Sự khác biệt giữa các tab là gì?

**Trả lời:**

3 tab dùng chung cấu trúc state và flow, chỉ khác ở **bước generate**:

**State chung của cả 3 tab:**
```ts
const [words, setWords] = useState<ExtendedWord[]>([])
const [selectedId, setSelectedId] = useState<number | null>(null)
const [editingId, setEditingId] = useState<number | null>(null)
const [previewResult, setPreviewResult] = useState<any>(null)
```

**Điểm khác nhau — mutation để generate:**
- `AiGenerateFromTextTab`: `useExtractFromText()` — phân tích đoạn văn, extract từ vựng có trong text
- `AiGenerateFromTopicTab`: `useExtractFromTopic({ topic, count: 25 })` — generate 25 từ theo chủ đề
- `AiGenerateFromImageTab`: `useExtractFromImage()` — OCR/phân tích ảnh, extract từ vựng

**Flow sau generate là giống nhau 100%:**
1. AI trả về danh sách `AiNote[]` → map thành `ExtendedWord[]` với `id` và `templateId`
2. User review, chỉnh sửa inline qua `WordItem`
3. Bấm "Preview" → `usePreviewNotes` — server tạo audio TTS, trả về `summary` (cache hit/miss)
4. Bấm "Lưu" → `useConfirmNotes` — gửi toàn bộ payload lên server

**Component tái sử dụng:** `WordItem` và `CardPreview` được dùng ở cả 3 tab. Điều này có nghĩa là logic chỉnh sửa, upload ảnh, chọn template đều centralized.

---

### Câu 11: `previewMutation` (usePreviewNotes) — bước preview audio trước khi lưu có ý nghĩa gì?

**Trả lời:**

```ts
const handlePreview = async () => {
  const res = await previewMutation.mutateAsync({
    words: words.map((w) => ({ word: w.word, detectedLanguage: languageId })),
  })
  setPreviewResult(res?.data?.data ?? res)
}

// Summary hiển thị sau preview:
const previewSummary = previewResult?.summary
// { cacheHit: 3, cacheMiss: 7, unsupportedLanguage: 0 }
```

Preview step yêu cầu server **chuẩn bị audio TTS** cho tất cả từ:
- `cacheHit`: từ đã có audio trong cache → dùng lại ngay
- `cacheMiss`: từ mới → server sẽ generate TTS khi confirm
- `unsupportedLanguage`: ngôn ngữ không có TTS → không có audio

Bước này có 2 lợi ích: (1) User biết trước audio nào sẽ được tạo mới, tránh bất ngờ. (2) Server có thể warm up cache trước — khi confirm thực sự, audio đã sẵn sàng, không phải chờ.

UI chuyển sang trạng thái "đã preview" với 2 nút Hủy/Lưu thay thế nút Preview:
```tsx
{previewResult ? (
  <div className="flex gap-3">
    <Button onClick={handleCancelPreview}>Hủy</Button>
    <Button onClick={handleSave}>Lưu</Button>
  </div>
) : (
  <Button onClick={handlePreview}>Preview</Button>
)}
```

---

### Câu 12: `WordItem` — inline editing hoạt động như thế nào? Tại sao state editing được lift lên component cha?

**Trả lời:**

`WordItem` nhận editing state từ cha và gọi callbacks thay vì tự manage:

```tsx
// WordItem không có local state cho editing — nhận từ props
{isEditing && editField === "word" ? (
  <InlineEdit
    value={editValue}
    onChange={onEditValueChange}    // Callback lên cha
    onSave={onSaveEdit}            // Callback lên cha
    onCancel={onCancelEdit}
  />
) : (
  <span onClick={(e) => { e.stopPropagation(); onStartEdit("word") }}>
    {word.word}
  </span>
)}
```

**Tại sao lift state lên cha:** Tại một thời điểm chỉ có thể có **1 word đang edit** (editingId) và **1 field đang edit** (editField). Nếu mỗi `WordItem` tự quản lý editing state, sẽ không có cơ chế để cancel editing của item A khi user click vào item B.

Với state ở cha:
```ts
// Cha quản lý
const [editingId, setEditingId] = useState<number | null>(null)
const [editField, setEditField] = useState<EditField>("word")
const [editValue, setEditValue] = useState("")

const startEdit = (id: number, field: EditField) => {
  const w = words.find((w) => w.id === id)
  setEditingId(id)
  setEditField(field)
  setEditValue((w as any)[field] ?? "")
}

const saveEdit = () => {
  setWords((prev) => prev.map((w) =>
    w.id === editingId ? { ...w, [editField]: editValue } : w
  ))
  setEditingId(null)
}
```

`saveEdit` dùng dynamic key `[editField]` để update đúng field (word, meaning, ipa, hoặc example) mà không cần 4 hàm riêng biệt.

---

### Câu 13: `WordItem` — tại sao `e.stopPropagation()` được gọi ở nhiều chỗ?

**Trả lời:**

`WordItem` có `onClick` ở cấp card để select item (`onSelect`). Nhưng bên trong có nhiều interactive element nhỏ hơn — inline edit spans, Select dropdowns, Upload button, Delete button — mỗi cái có logic riêng.

Nếu không có `stopPropagation`, click vào bất kỳ element con nào cũng sẽ bubble lên card → trigger `onSelect` → thay đổi `selectedId`. Điều này gây conflict:

```tsx
// Ví dụ: Click vào Select để đổi template
<div onClick={(e) => e.stopPropagation()}>   // Wrapper chặn bubble
  <Select value={word.templateId} onValueChange={onTemplateChange}>
    ...
  </Select>
</div>
```

Tương tự với inline edit spans:
```tsx
<span onClick={(e) => { e.stopPropagation(); onStartEdit("word") }}>
  {word.word}
</span>
```

Click `span` → `stopPropagation` → chỉ trigger `onStartEdit("word")`, không trigger `onSelect`. Nếu thiếu `stopPropagation`, UI sẽ vừa select item vừa start edit — không phải behavior mong muốn (vì chỉ click vào item đã selected mới bắt đầu edit).

---

## PHẦN 5 — STUDY SESSION DETAIL

---

### Câu 14: `StudySessionDetailPage` hiển thị rating distribution — logic tính `pct` và render progress bar như thế nào?

**Trả lời:**

```tsx
{(["AGAIN", "HARD", "GOOD", "EASY"] as const).map((r) => {
  const count = stats[r.toLowerCase() as keyof typeof stats] as number
  const total = stats.reviewedCards || 1  // Tránh chia cho 0
  const pct = Math.round((count / total) * 100)
  const info = RATING_LABELS[r]  // { label, color, bg }

  return (
    <div key={r} className={`rounded-xl ${info.bg} p-3 text-center`}>
      <p className={`text-lg font-black ${info.color}`}>{count}</p>
      <p className={`text-xs font-semibold ${info.color}`}>{info.label}</p>
      <div className="mt-1.5 h-1.5 w-full overflow-hidden rounded-full bg-white/60">
        <div
          className={`h-full rounded-full ${info.color.replace("text-", "bg-")}`}
          style={{ width: `${pct}%` }}
        />
      </div>
    </div>
  )
})}
```

`stats.reviewedCards || 1` — guard tránh division by zero nếu session chưa có card nào.

`info.color.replace("text-", "bg-")` — trick tái sử dụng color class: nếu `info.color = "text-red-500"` thì progress bar dùng `bg-red-500`. Không cần define thêm `bgColor` riêng trong `RATING_LABELS`.

---

### Câu 15: Review log hiển thị word/meaning khác nhau tùy `templateType`. Logic xử lý 3 loại card ra sao?

**Trả lời:**

```ts
const templateType = log.card?.note?.template?.type

const word = templateType === "CLOZE"
  ? (log.card?.note?.fields as any)?.Text || "Cloze"      // Field Text của cloze
  : templateType === "IMAGE_OCCLUSION"
    ? `Occlusion #${(log.card?.variantIndex ?? 0) + 1}`   // "Occlusion #1", "#2"...
    : log.card?.note?.word                                  // Vocabulary word

const meaning = templateType === "CLOZE"
  ? (log.card?.note?.fields as any)?.Extra || "—"         // Field Extra của cloze
  : templateType === "IMAGE_OCCLUSION"
    ? (log.card?.note?.occlusionMasks as any[])
        ?.map((m: any) => m.label).filter(Boolean).join(", ") || "—"  // Labels của masks
    : log.card?.note?.meaning                              // Meaning của vocabulary
```

3 loại card có cấu trúc data khác nhau hoàn toàn:
- **Vocabulary card:** `note.word` + `note.meaning`
- **Cloze card:** `note.fields.Text` (câu text đầy đủ với cloze syntax) + `note.fields.Extra`
- **Image Occlusion card:** Label số thứ tự variant + tất cả mask labels join bằng dấu phẩy

Đây là discriminated rendering pattern — không có polymorphism hay component map, mà dùng ternary chain vì logic chỉ cần cho phần display text nhỏ, không phức tạp đủ để tách component riêng.

---

### Câu 16: `state transition` trong review log — `stateBefore → stateAfter` có ý nghĩa gì trong spaced repetition?

**Trả lời:**

```tsx
<span className="text-xs text-text-muted">
  {STATE_LABELS[log.stateBefore] ?? log.stateBefore} →
  {STATE_LABELS[log.stateAfter] ?? log.stateAfter}
</span>
```

State trong spaced repetition (FSRS algorithm) đại diện cho giai đoạn học của card:

- **New**: Card chưa từng được ôn
- **Learning**: Card đang trong giai đoạn học ban đầu (interval tính bằng phút/giờ)
- **Review**: Card đã "thuộc", interval tính bằng ngày/tuần
- **Relearning**: Card đã quên sau khi đã thuộc (rating AGAIN khi đang Review)

Transition `New → Learning` = lần đầu tiên ôn. `Learning → Review` = đã nắm được cơ bản. `Review → Relearning` = quên bài. `Relearning → Review` = học lại thành công.

Hiển thị transition này trong study session detail cho user thấy rõ card nào đang tiến bộ và card nào bị regress — thông tin quan trọng để đánh giá hiệu quả buổi học.

---

## PHẦN 6 — CREATE NOTES PAGE ARCHITECTURE

---

### Câu 17: `CreateNotePage` có 5 tabs. Tab switching được implement như thế nào? Tại sao không dùng URL params?

**Trả lời:**

```ts
const TABS = [
  { key: "manual", label: "Manual", icon: BookOpen },
  { key: "csv", label: "Import CSV", icon: FileSpreadsheet },
  { key: "ai-generate-from-text", ... },
  { key: "ai-generate-from-topic", ... },
  { key: "ai-generate-from-image", ... },
] as const

type TabKey = (typeof TABS)[number]["key"]  // Union type từ array

const [activeTab, setActiveTab] = useState<TabKey>("manual")
```

Tab state dùng `useState` local thay vì URL params vì:

**State là ephemeral:** Khi user switch tab, toàn bộ state của tab cũ (words list, preview result, text input...) bị reset. Đây là intentional — nếu user đang AI generate từ text rồi switch sang manual tab, words list không cần persist.

**Không có shareable URL:** Không có use case nào cần share link đến một tab cụ thể của create-notes page.

Conditional rendering thay vì hidden/show để đảm bảo state reset:
```tsx
{activeTab === "manual" && <ManualTab deckId={deckId} />}
{activeTab === "ai-generate-from-text" && <AiGenerateFromTextTab deckId={deckId} />}
```

Khi `activeTab` thay đổi, component cũ unmount hoàn toàn (state reset), component mới mount với state fresh.

---

### Câu 18: `templates` được filter bỏ CLOZE và IMAGE_OCCLUSION trong AI generate tabs. Tại sao?

**Trả lời:**

```ts
const templates = useMemo(
  () => allTemplates.filter(
    (t) => t.type !== NoteTemplateType.CLOZE && t.type !== NoteTemplateType.IMAGE_OCCLUSION
  ),
  [allTemplates]
)
```

AI generate flow tạo vocabulary cards từ word + meaning + IPA + example. Cloze và Image Occlusion là loại card hoàn toàn khác về cấu trúc data:

- **Cloze** cần field `Text` với cloze syntax `{{c1::...}}` — AI không generate ra dạng này
- **Image Occlusion** cần `imageUrl` + `masks[]` — AI không generate ra dạng này

Nếu user chọn template Cloze cho một AI-generated word, payload gửi lên (`word`, `meaning`, `ipa`...) sẽ không match structure mà Cloze template expect → card render sai.

Filter này là UX guard — chỉ show những template compatible với data AI generate. Cloze và Image Occlusion có UI riêng (`ManualTab` với `ClozeEditor` và `ImageOcclusionEditor`).

---

### Câu 19: `handleSave` validation — kiểm tra `!w.templateId` trước khi submit. Tại sao cần bước này?

**Trả lời:**

```ts
const handleSave = async () => {
  const invalid = words.filter((w) => !w.templateId)
  if (invalid.length) {
    toast.error(`Có ${invalid.length} từ chưa chọn template`)
    return
  }
  // ...
}
```

Mỗi word cần được assign một template để server biết cách tạo card (frontHtml/backHtml). Không có template → server không thể tạo card → API error.

Vấn đề xảy ra khi `defaultTemplateId = templates[0]?.id ?? ""`. Nếu `templates` array rỗng (server chưa return templates, hoặc filter xóa hết), `defaultTemplateId = ""` — tất cả words được init với `templateId: ""` (empty string, falsy).

Guard check `!w.templateId` catch cả `""` lẫn `null`/`undefined`, đưa ra thông báo lỗi rõ ràng thay vì để API fail với error message khó hiểu.

---

*Tổng cộng: 19 câu hỏi & trả lời cho Recalio*
*File nguồn: `image-occlusion-editor.tsx`, `image-occlusion-card-view.tsx`, `cloze-editor.tsx`, `card-preview.tsx` (×2), `word-item.tsx`, `ai-generate-from-text-tab.tsx`, `ai-generate-from-topic-tab.tsx`, `ai-generate-from-image-tab.tsx`, `page.tsx` (create-notes), `page.tsx` (study-session-detail), `review-card.tsx`*
