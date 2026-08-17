# Câu hỏi & Trả lời phỏng vấn — BingeBox
> Dựa trực tiếp vào code thực tế: `useSeatSocket.ts`, `useSeatStore.ts`, `seat-list.tsx` (booking), `seat-list.tsx` (admin), `seat-editor.tsx`, `seat-item.tsx`, `page.tsx` (booking), `order-summary.tsx`, `food-list.tsx`, `voucher-list.tsx`, `user-point.tsx`

---

## PHẦN 1 — REAL-TIME SEAT MAP (Socket.IO)

---

### Câu 1: Seat map realtime qua Socket.IO — khi 2 user cùng chọn 1 ghế, xử lý conflict như thế nào?

**Trả lời:**

Conflict resolution xử lý hoàn toàn ở **server-side**. Client không tự quyết định ghế có available hay không — nó chỉ phản ánh những gì server broadcast xuống.

**Flow cụ thể từ `useSeatSocket.ts`:**

Khi component mount, client join vào một "room" theo `showtimeId`:
```ts
socket.emit("join-showtime", showtimeId);
```

Khi user click ghế, `seat-list.tsx` gọi:
```ts
holdSeat(showtimeId, seat._id);
// tức là: socket.emit("hold-seat", { showtimeId, seatId })
```

Server nhận `hold-seat`, kiểm tra DB/Redis xem ghế còn `AVAILABLE` không:
- **Còn available:** cập nhật trạng thái thành `HOLD`, broadcast `seat:held` đến toàn bộ user trong room `showtimeId`
- **Đã bị hold/sold:** không làm gì hoặc emit error về đúng user đó

Client lắng nghe broadcast và **update thẳng vào TanStack Query cache** — không cần refetch:
```ts
const handleSeatHeld = ({ seatId }: { seatId: string }) => {
  queryClientRef.current.setQueryData<any>(
    [...SEAT_QUERY_KEY, "by-showtime", showtimeId],
    (old: any) => ({
      ...old,
      data: old.data.map((seat: any) =>
        seat._id === seatId ? { ...seat, status: "HOLD" } : seat
      ),
    })
  );
};
socket.on("seat:held", handleSeatHeld);
```

Kết quả: User B đang xem cùng suất chiếu sẽ thấy ghế chuyển màu sang `HOLD` ngay lập tức mà không cần reload. User A nếu bị từ chối thì ghế không đổi màu (server không broadcast về A).

**Về ghế đôi (couple seat):** `seat-list.tsx` emit hold cho cả 2 ghế cùng lúc:
```ts
holdSeat(showtimeId, seat._id);
holdSeat(showtimeId, partnerSeat._id);
```
Và check trước khi hold — nếu partner không còn `AVAILABLE` thì toast warning và abort:
```ts
if (!partnerSeat || partnerSeat.status !== "AVAILABLE") {
  return toast.warning("Ghế đôi này không khả dụng đủ cặp");
}
```

---

### Câu 2: `joinedRef` trong `useSeatSocket` dùng để làm gì? Tại sao không chỉ emit join trong useEffect bình thường?

**Trả lời:**

`joinedRef` ngăn emit `join-showtime` nhiều lần khi component re-render:

```ts
const joinedRef = useRef(false);

if (!joinedRef.current) {
  socket.emit("join-showtime", showtimeId);
  joinedRef.current = true;
}
```

`useEffect` với dependency `[showtimeId]` sẽ re-run nếu `showtimeId` thay đổi — điều đó ổn. Nhưng trong **React Strict Mode** (dev), `useEffect` chạy 2 lần khi mount để detect side effects. Nếu không có `joinedRef`, user sẽ join room 2 lần, server có thể tính là 2 connection riêng biệt và duplicate event.

`ref` không trigger re-render nên là lựa chọn đúng cho loại flag "đã làm rồi, đừng làm nữa" này. Nếu dùng `useState` thay cho `ref`, mỗi lần set flag sẽ trigger thêm một re-render không cần thiết.

Tương tự, `queryClientRef` được dùng thay vì đưa `queryClient` vào dependency array:
```ts
const queryClientRef = useRef(queryClient);
queryClientRef.current = queryClient;
```
Pattern này cho phép handler `handleSeatHeld` luôn truy cập `queryClient` mới nhất mà không cần recreate listener mỗi render — tránh `socket.off` / `socket.on` liên tục gây memory leak hoặc mất event.

---

### Câu 3: Khi user rời trang booking (unmount), ghế đang giữ được release như thế nào?

**Trả lời:**

Trong `seat-list.tsx`, cleanup function của `useEffect` gọi release cho toàn bộ ghế đang được chọn:

```ts
const selectedRef = useRef(selectedSeats);
selectedRef.current = selectedSeats;

useEffect(() => {
  return () => {
    selectedRef.current.forEach((s) => releaseSeat(showtimeId, s._id));
  };
}, []); // dependency rỗng → chỉ cleanup khi unmount
```

**Tại sao phải dùng `selectedRef` thay vì đưa `selectedSeats` vào dependency:**

Nếu đưa `selectedSeats` vào dependency:
```ts
useEffect(() => {
  return () => { /* cleanup */ };
}, [selectedSeats]); // ❌ sai
```
Cleanup sẽ chạy **mỗi lần user chọn hoặc bỏ chọn ghế**, không chỉ khi unmount. Điều này khiến ghế bị release sai lúc — ngay sau khi user vừa hold xong.

Với `ref`, cleanup chỉ chạy **một lần duy nhất khi component unmount**, nhưng `selectedRef.current` lúc đó đang giữ giá trị mới nhất của `selectedSeats` (do được update mỗi render: `selectedRef.current = selectedSeats`).

`releaseSeat` emit `socket.emit("release-seat", { showtimeId, seatId })` → server nhận, set ghế về `AVAILABLE`, broadcast `seat:released` → các user khác thấy ghế available lại ngay lập tức.

---

### Câu 4: `useSeatSocket` cleanup — tại sao `socket.off` dùng reference đến đúng handler function thay vì tên event?

**Trả lời:**

```ts
socket.on("seat:held", handleSeatHeld);
socket.on("seat:released", handleSeatReleased);

return () => {
  socket.off("seat:held", handleSeatHeld);
  socket.off("seat:released", handleSeatReleased);
};
```

Nếu chỉ gọi `socket.off("seat:held")` không truyền reference function, Socket.IO sẽ remove **tất cả** listeners của event đó — kể cả những listener được đăng ký bởi component khác. Truyền đúng reference function đảm bảo chỉ remove đúng listener của hook này, không ảnh hưởng listener nào khác.

Đây cũng là lý do tại sao `handleSeatHeld` và `handleSeatReleased` được định nghĩa bên trong `useEffect` chứ không phải ngoài — để đảm bảo mỗi lần effect chạy lại (nếu `showtimeId` thay đổi), reference mới được tạo và cleanup đúng reference đó.

---

## PHẦN 2 — BOOKING FLOW (3 BƯỚC)

---

### Câu 5: 3-step booking flow — state được quản lý ở đâu? Tại sao không dùng Zustand?

**Trả lời:**

Toàn bộ state của booking flow được lift lên `BookingDetailPage` (`page.tsx`) dưới dạng `useState` thuần:

```ts
const [step, setStep] = useState(1);           // Bước hiện tại: 1 | 2 | 3
const [selectedSeats, setSelectedSeats] = useState<any[]>([]);
const [selectedFoods, setSelectedFoods] = useState<any[]>([]);
const [selectedVoucher, setSelectedVoucher] = useState<any>(null);
const [pointsUsed, setPointsUsed] = useState(0);
```

State và setter được pass xuống từng component con theo đúng step:
- **Step 1** → `<SeatListClient selectedSeats setSelectedSeats />`
- **Step 2** → `<FoodListClient selectedFoods setSelectedFoods />`
- **Step 3** → `<UserPoint />` + `<VoucherListClient />`
- **Mọi step** → `<OrderSummary />` nhận tất cả để tính tiền và submit

**Tại sao không dùng Zustand:** Booking flow chỉ tồn tại trong 1 route `/booking/[id]`, không cần share sang route khác. `useState` + props đủ và đơn giản hơn. Nếu dùng Zustand, cần nhớ reset store thủ công khi user rời trang, nếu không lần sau vào trang booking sẽ thấy state cũ (ghế đã chọn từ lần trước).

**Back về step trước không mất data:** Vì state sống ở `BookingDetailPage`, chuyển `step` chỉ thay đổi component *nào được render* — không unmount `BookingDetailPage`. Ghế đã chọn ở step 1 vẫn còn trong `selectedSeats` khi quay lại từ step 2.

---

### Câu 6: `previewPrice` trong seat-list hoạt động như thế nào? Tại sao cần nó?

**Trả lời:**

Sau khi user chọn ghế, giá hiển thị ban đầu lấy từ `seat.seatType?.price` — đây là giá static của loại ghế. Tuy nhiên giá thực tế của vé có thể khác tùy suất chiếu (premiere, ngày lễ, giờ vàng có giá riêng).

`previewPrice` gọi API để lấy giá chính xác cho cặp `(seatId, showtimeId)`. Em dùng **optimistic approach**: thêm ghế vào list ngay lập tức với giá tạm, rồi update giá thật khi API trả về:

```ts
// 1. Thêm ghế ngay với giá tạm
setSelectedSeats([...selectedSeats,
  { ...seat, price: seat.seatType?.price || 0 }
]);

// 2. Gọi API lấy giá thật (không block UI)
previewPrice({ seatId: seat._id, showtimeId }, {
  onSuccess: (res) => {
    const priceFromServer = res.data.data.price;
    setSelectedSeats((prev) =>
      prev.map((s) =>
        s._id === seat._id ? { ...s, price: priceFromServer } : s
      )
    );
  },
  onSettled: () => {
    setLoadingSeats((prev) => {
      const next = new Set(prev);
      next.delete(seat._id);
      return next;
    });
  }
});
```

`loadingSeats` là `Set<string>` chứa các `seatId` đang chờ price confirm. Trong thời gian đó, `toggleSeat` lock toàn bộ interaction:
```ts
if (loadingSeats.size > 0) return;
```
Ngăn user chọn ghế khác trong khi giá chưa được confirm — tránh race condition giữa các concurrent preview requests.

---

### Câu 7: Tính giá trong OrderSummary — logic discount voucher và điểm thưởng hoạt động như thế nào?

**Trả lời:**

Từ `order-summary.tsx`, công thức tính theo đúng 5 bước:

```ts
// 1. Tổng tiền ghế — fallback qua 3 nguồn giá
const totalSeatsPrice = selectedSeats.reduce((sum, seat) =>
  sum + (seat.price || seat.seatType?.price || showtime?.price || 0), 0);

// 2. Tổng tiền đồ ăn
const totalFoodsPrice = selectedFoods.reduce((sum, food) =>
  sum + food.price * food.quantity, 0);

// 3. Subtotal trước khi giảm
const subTotal = totalSeatsPrice + totalFoodsPrice;

// 4. Discount voucher — cap bởi maxDiscountAmount
const discount = selectedVoucher
  ? Math.min(selectedVoucher.maxDiscountAmount, subTotal)
  : 0;

// 5. Final = subTotal - discount - points, không được âm
const finalTotal = Math.max(0, subTotal - discount - (pointsUsed || 0));
```

Hai điểm kỹ thuật đáng chú ý:

`Math.min(voucher.maxDiscountAmount, subTotal)` — discount không thể vượt quá tổng đơn hàng. Ví dụ voucher giảm tối đa 200k nhưng đơn chỉ 150k → discount chỉ là 150k, không phải 200k.

`Math.max(0, ...)` — finalTotal không thể âm dù pointsUsed lớn hơn số tiền còn lại.

UI còn hiển thị warning inline khi chưa đủ điều kiện áp voucher (đơn chưa đủ `minOrderValue`):
```tsx
{subTotal < selectedVoucher.minOrderValue && (
  <span className="text-red-500 text-[10px]">
    * Chưa đủ điều kiện (tối thiểu {selectedVoucher.minOrderValue.toLocaleString()} đ)
  </span>
)}
```

---

### Câu 8: Validation và submit trong `OrderSummary` — flow xử lý từng bước như thế nào?

**Trả lời:**

`handleNext` trong `order-summary.tsx` xử lý logic khác nhau theo từng step:

```ts
const handleNext = () => {
  // Step 1: bắt buộc phải chọn ít nhất 1 ghế
  if (step === 1) {
    if (selectedSeats.length === 0) {
      return toast.error("Vui lòng chọn ít nhất một ghế");
    }
    setStep(2);
    return;
  }

  // Step 2: không bắt buộc — skip thẳng
  if (step === 2) {
    setStep(3);
    return;
  }

  // Step 3: build payload và gọi API
  if (step === 3) {
    const bookingPayload = {
      showtimeId: showtime._id,
      seatIds: selectedSeats.map((s) => s._id),
      foods: selectedFoods.map((f) => ({ foodId: f._id, quantity: f.quantity })),
      voucherCode: selectedVoucher?.code || "",
      pointsUsed: pointsUsed,
    };

    createBooking(bookingPayload, {
      onSuccess: (res) => {
        const bookingId = res.data?.data._id;
        router.push(`/payment/${bookingId}`); // Redirect sang trang thanh toán
      },
      onError: (error) => handleError(error),
    });
  }
};
```

Đồ ăn và voucher/điểm thưởng là optional — không có validation bắt buộc ở step 2 và 3. Chỉ ghế là mandatory vì không có ghế thì không thể tạo booking.

Button bị disable khi `isPending` để tránh double-submit, đồng thời hiển thị spinner.

---

### Câu 9: `UserPoint` — tại sao cần validate cả 2 điều kiện `numValue > currentPoints` và `numValue > finalAmountBeforePoints`?

**Trả lời:**

Từ `user-point.tsx`, hai ràng buộc bảo vệ 2 edge case hoàn toàn khác nhau:

```ts
if (numValue > currentPoints) {
  // Guard 1: không thể dùng điểm nhiều hơn số điểm đang có
  setPointsUsed(currentPoints);
  setInputValue(currentPoints.toString());
} else if (numValue > finalAmountBeforePoints) {
  // Guard 2: không thể dùng điểm vượt quá tổng tiền đơn
  setPointsUsed(finalAmountBeforePoints);
  setInputValue(finalAmountBeforePoints.toString());
} else {
  setPointsUsed(numValue);
}
```

**Guard 1** — ràng buộc từ phía tài khoản: user có 100,000 điểm, không thể nhập 200,000.

**Guard 2** — ràng buộc từ phía đơn hàng: user có 500,000 điểm, đơn chỉ 200,000đ. Nếu dùng 300,000 điểm thì `finalTotal` sẽ âm (-100,000), không hợp lệ về mặt business. Cap thứ 2 đảm bảo điểm dùng ≤ tổng đơn còn lại sau khi trừ voucher.

Nút "TỐI ĐA" implement chính xác điều này bằng `Math.min` của cả hai giới hạn:
```ts
const handleUseMax = () => {
  const maxPossible = Math.min(currentPoints, finalAmountBeforePoints);
  setPointsUsed(maxPossible);
};
```

Ngoài ra, input chỉ cho phép nhập số bằng cách strip ký tự không phải số:
```ts
const value = e.target.value.replace(/\D/g, "");
```

---

### Câu 10: `FoodListClient` — tại sao không dùng Zustand hay TanStack mutation để quản lý food quantity?

**Trả lời:**

Trong `food-list.tsx`, `selectedFoods` là prop được lift lên `BookingDetailPage`. Logic `updateQuantity` chỉ làm việc với state này thuần túy:

```ts
const updateQuantity = (food: any, delta: number) => {
  const existed = selectedFoods.find((f) => f._id === food._id);
  if (existed) {
    const newQty = existed.quantity + delta;
    if (newQty <= 0) {
      // Về 0 → xóa khỏi list
      setSelectedFoods(selectedFoods.filter((f) => f._id !== food._id));
    } else {
      setSelectedFoods(selectedFoods.map((f) =>
        f._id === food._id ? { ...f, quantity: newQty } : f
      ));
    }
  } else if (delta > 0) {
    // Chưa có trong list → thêm mới với quantity = 1
    setSelectedFoods([...selectedFoods, { ...food, quantity: 1 }]);
  }
};
```

Lý do không cần Zustand hay mutation: food selection là **ephemeral UI state** — nó chỉ tồn tại cho đến khi user submit booking. Không cần persist, không cần share sang route khác, không cần sync với server cho đến bước cuối.

Chỉ khi user bấm "Hoàn thành" ở step 3, toàn bộ `selectedFoods` mới được đưa vào payload một lần duy nhất:
```ts
foods: selectedFoods.map((f) => ({ foodId: f._id, quantity: f.quantity }))
```

Dùng mutation để save food quantity trung gian lên server sẽ tốn nhiều API call không cần thiết và phức tạp hóa flow.

---

## PHẦN 3 — ADMIN SEAT MAP EDITOR

---

### Câu 11: Admin seat map editor — `useSeatStore` được thiết kế như thế nào? Tại sao dùng Zustand ở đây trong khi booking flow không dùng?

**Trả lời:**

`useSeatStore` quản lý toàn bộ state và business logic của editor:

```ts
interface SeatStore {
  rows: any[];   // Mảng hàng, mỗi hàng: { rowKey: string, seats: SeatConfig[] }
  selectedSeat: { rowIndex: number; seatIndex: number } | null;

  // Khởi tạo
  setInitialRows: (seats, categories) => void;
  initDefaultLayout: (defaultCategory) => void;

  // Selection
  setSelectedSeat: (selected) => void;

  // Chỉnh sửa ghế
  changeSeatType: (categoryId, categories) => void;
  toggleCoupleSeat: (categories) => void;
  insertSeat: (isBlocked, side) => void;
  removeSeat: () => void;

  // Chỉnh sửa hàng
  addRowBelow: (rowIndex) => void;
  removeRow: (rowIndex) => void;
}
```

**Tại sao Zustand ở đây nhưng không dùng ở booking flow:**

Seat map editor có **3 component độc lập** cùng truy cập và mutate state:
- `SeatList` — render toàn bộ grid, handle add/remove row
- `SeatItem` (admin) — render từng ghế, biết ghế nào đang `active`
- `SeatEditor` — panel chỉnh sửa ghế được chọn (changeSeatType, toggleCoupleSeat, insertSeat, removeSeat)

Nếu dùng props drilling, `SeatList` phải nhận và forward 8+ props/callbacks xuống `SeatEditor`. Với Zustand, `SeatEditor` gọi thẳng store — không cần trung gian:

```ts
// SeatEditor.tsx — zero props liên quan đến state
const { rows, selectedSeat, changeSeatType, toggleCoupleSeat,
        insertSeat, removeSeat } = useSeatStore();
```

Booking flow chỉ có 1 "owner" duy nhất là `BookingDetailPage` pass state xuống các bước — props drilling còn hợp lý. Editor có 3 component ngang hàng cùng cần mutate → Zustand là lựa chọn đúng.

---

### Câu 12: `toggleCoupleSeat` trong `useSeatStore` — khi ghép 2 ghế thành couple, logic xử lý như thế nào?

**Trả lời:**

`toggleCoupleSeat` xử lý 2 chiều bật/tắt từ `useSeatStore.ts`:

**Khi TẠO couple (ghế hiện tại chưa phải couple):**
```ts
const nextSeat = newSeats[seatIndex + 1];

// Validation trước khi ghép
if (!nextSeat || nextSeat.isBlocked || nextSeat.isCoupleSeat) {
  toast.error("Cần một ghế trống bên phải để tạo ghế đôi!");
  return;
}

// Update cả 2 ghế: trỏ partnerSeatCode vào nhau
newSeats[seatIndex] = {
  ...seat,
  isCoupleSeat: true,
  partnerSeatCode: nextSeat.code,
  seatType: coupleCategory ?? seat.seatType   // Tự động đổi sang loại "ghế đôi"
};
newSeats[seatIndex + 1] = {
  ...nextSeat,
  isCoupleSeat: true,
  partnerSeatCode: seat.code,
  seatType: coupleCategory ?? seat.seatType
};
```

**Khi HỦY couple:**
```ts
const partnerCode = seat.partnerSeatCode;
currentRow.seats = newSeats.map(s => {
  if (s.code === seat.code || s.code === partnerCode) {
    return {
      ...s,
      isCoupleSeat: false,
      partnerSeatCode: undefined,
      seatType: defaultCat  // Reset về loại ghế mặc định
    };
  }
  return s;
});
```

Cả hai ghế trong cặp đều được xử lý cùng lúc — không thể có trạng thái một ghế là couple còn ghế kia thì không.

`coupleCategory` được tìm bằng cách filter category có tên chứa "đôi" hoặc có flag `isCoupleType`:
```ts
const coupleCategory = categories.find((c) =>
  c.name.toLowerCase().includes("đôi") || c.isCoupleType === true
);
```

---

### Câu 13: `reindexAll` và `syncRowData` — hai hàm này giải quyết vấn đề gì? Khi nào dùng cái nào?

**Trả lời:**

Khi admin thêm/xóa hàng hoặc ghế, mã ghế (A1, B2...) và `partnerSeatCode` của couple seats có thể bị sai. Hai helper này đảm bảo tính nhất quán:

**`reindexAll`** — dùng sau khi thao tác trên *hàng* (`addRowBelow`, `removeRow`):
```ts
const reindexAll = (allRows: any[]) => {
  return allRows.map((row, rIdx) => {
    const newRowKey = String.fromCharCode(65 + rIdx); // 0→A, 1→B, 2→C...
    let seatCount = 1;
    const newSeats = row.seats.map((s) => ({
      ...s,
      row: newRowKey,
      code: s.isBlocked ? "TRỐNG" : `${newRowKey}${seatCount++}`
    }));
    // Sau khi renumber, fix partnerSeatCode cho couple seats
    const finalSeats = newSeats.map((s, sIdx) => {
      if (s.isCoupleSeat) {
        const next = newSeats[sIdx + 1];
        const prev = newSeats[sIdx - 1];
        if (next?.isCoupleSeat) return { ...s, partnerSeatCode: next.code };
        if (prev?.isCoupleSeat) return { ...s, partnerSeatCode: prev.code };
      }
      return s;
    });
    return { ...row, rowKey: newRowKey, seats: finalSeats };
  });
};
```

**`syncRowData`** — dùng sau khi thao tác trên *ghế trong 1 hàng* (`insertSeat`, `removeSeat`):
- Chỉ renumber trong phạm vi 1 hàng, không đụng `rowKey`
- Fix `partnerSeatCode` cho couple seats trong hàng đó

**Khi nào dùng cái nào:**

| Thao tác | Helper |
|---|---|
| `addRowBelow` | `reindexAll` — cần cập nhật rowKey toàn bộ |
| `removeRow` | `reindexAll` — cần cập nhật rowKey toàn bộ |
| `insertSeat` | `syncRowData` — chỉ cần sync 1 hàng |
| `removeSeat` | `syncRowData` — chỉ cần sync 1 hàng |

Ví dụ: Hàng A có `[A1, A2(couple), A3(couple), A4]`. Admin xóa A2 → `syncRowData` chạy → kết quả `[A1, A2, A3]`. Vì couple bị xóa một nửa, A2 và A3 mới không còn là couple nữa.

---

### Câu 14: `insertSeat` — tại sao cần check trường hợp chèn vào giữa ghế đôi?

**Trả lời:**

Từ `useSeatStore.ts`:
```ts
const nextSeat = rows[rowIndex].seats[seatIndex + 1];
const prevSeat = rows[rowIndex].seats[seatIndex - 1];

if (side === "right" && seat.isCoupleSeat && nextSeat?.code === seat.partnerSeatCode) {
  toast.error("Không thể chèn vào giữa ghế đôi!");
  return;
}
if (side === "left" && seat.isCoupleSeat && prevSeat?.code === seat.partnerSeatCode) {
  toast.error("Không thể chèn vào giữa ghế đôi!");
  return;
}
```

Couple seat là một cặp liền kề bất khả phân — nếu chèn ghế vào giữa, cặp đôi sẽ bị tách ra thành 2 ghế thường không liền nhau. Về mặt UI phòng chiếu, ghế đôi phải kề nhau mới có thể render đúng (visual joined seat). Guard này bảo vệ tính toàn vẹn của dữ liệu.

---

### Câu 15: `handleSave` trong admin `SeatList` — data được format như thế nào trước khi gửi lên server?

**Trả lời:**

```ts
const handleSave = () => {
  const formattedSeats = rows.flatMap((row) => {
    let currentColumn = 1;
    return row.seats.map((seat) => {
      const seatPayload: any = {
        code: seat.code,
        row: row.rowKey,
        column: currentColumn++,          // Gán column tự động theo thứ tự
        isBlocked: seat.isBlocked,
        seatTypeId: seat.seatType?._id || seat.seatTypeId,
      };
      if (seat.isCoupleSeat) {
        seatPayload.isCoupleSeat = true;
        seatPayload.partnerSeatCode = seat.partnerSeatCode;
      }
      if (seat.isBlocked) {
        seatPayload.seatTypeId = undefined; // Ghế blocked không có seatType
      }
      return seatPayload;
    });
  });

  updateSeats({ roomId, data: { seats: formattedSeats } }, { ... });
};
```

`flatMap` flatten mảng 2D (rows → seats) thành mảng phẳng. `currentColumn` được tính tự động trong closure của mỗi row, không phụ thuộc vào `column` lưu trong store (vì admin có thể insert/delete ghế làm column trong store không còn chính xác).

---

## PHẦN 4 — SEAT ITEM & RENDERING

---

### Câu 16: `SeatItem` trong booking render màu ghế như thế nào? Tại sao dùng inline style thay vì Tailwind class?

**Trả lời:**

Từ `seat-item.tsx` (booking):

```ts
const STATUS_COLOR: Record<string, string> = {
  SOLD: "var(--color-sold)",
  HOLD: "var(--color-hold)",
};

const bgColor = isSelected
  ? "var(--color-selected)"
  : STATUS_COLOR[seat.status] || seat.seatType?.color || "#3f3f46";

return (
  <div
    style={{ backgroundColor: bgColor }}
    className={`w-10 h-10 ... ${isSold ? "grayscale" : ""}`}
  >
```

**Tại sao inline style thay vì Tailwind:** Màu `seat.seatType?.color` là giá trị **dynamic từ API** (ví dụ `#FFD700` cho VIP, `#FF6B6B` cho Premium). Tailwind purge CSS ở build time — các class như `bg-[#FFD700]` không được generate sẵn trừ khi được hardcode trong source. Inline style là cách duy nhất để apply màu dynamic từ server.

CSS variables (`var(--color-sold)`, `var(--color-selected)`) được define trong global CSS, đảm bảo consistent color cho các trạng thái cố định (sold, hold, selected) mà không hardcode hex vào component.

**Ghế đôi:** Tailwind class `rounded-none first:rounded-l-lg last:rounded-r-lg` tạo visual "joined" effect — chỉ bo góc ở đầu và cuối của cặp, không bo giữa:
```ts
seat.isCoupleSeat
  ? "rounded-none first:rounded-l-lg last:rounded-r-lg border-x border-white/10 w-20"
  : "rounded-md"
```
`w-20` (80px) cho ghế đôi so với `w-10` (40px) cho ghế thường.

---

### Câu 17: `groupRows` và `findPartnerSeat` trong seat-list hoạt động như thế nào?

**Trả lời:**

**`groupRows`** — transform mảng seat phẳng từ API thành cấu trúc theo hàng:
```ts
const groupRows = (seats: SeatItem[]) => {
  const grouped: Record<string, SeatItem[]> = {};
  seats.forEach((seat) => {
    if (!grouped[seat.row]) grouped[seat.row] = [];
    grouped[seat.row].push(seat);
  });
  return Object.keys(grouped)
    .sort()                   // Sắp xếp hàng theo alphabet: A, B, C...
    .map((rowKey) => ({
      rowKey,
      seats: grouped[rowKey].sort(
        (a, b) => (a.column || 0) - (b.column || 0)  // Sắp xếp ghế theo column
      ),
    }));
};
```

**`findPartnerSeat`** — tìm ghế đôi tương ứng dựa trên quy ước số chẵn/lẻ:
```ts
const findPartnerSeat = (allSeats: SeatItem[], seat: SeatItem) => {
  const seatNumber = parseInt(seat.code.replace(/[^\d]/g, ""), 10);
  const isEven = seatNumber % 2 === 0;
  const partnerCode = isEven
    ? seat.code.replace(/\d+$/, String(seatNumber - 1))  // A4 → A3
    : seat.code.replace(/\d+$/, String(seatNumber + 1)); // A3 → A4
  return allSeats.find(
    (s) => s.code === partnerCode && s.row === seat.row   // Cùng hàng
  );
};
```

Quy ước: ghế đôi luôn là cặp số lẻ + số chẵn liền kề (A1+A2, A3+A4...). Hàm tìm partner bằng cách flip số lẻ→chẵn và ngược lại, rồi filter cùng `row`.

---

*Tổng cộng: 17 câu hỏi & trả lời cho BingeBox*
*File nguồn: `useSeatSocket.ts`, `useSeatStore.ts`, `seat-list.tsx` (×2), `seat-editor.tsx`, `seat-item.tsx` (×2), `page.tsx`, `order-summary.tsx`, `food-list.tsx`, `voucher-list.tsx`, `user-point.tsx`, `room-info.tsx`*
