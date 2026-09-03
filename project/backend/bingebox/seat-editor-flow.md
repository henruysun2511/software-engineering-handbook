# TÀI LIỆU CHI TIẾT LOGIC SEAT EDITOR TRÊN FRONTEND (BINGEBOX ADMIN)

Tài liệu mô tả chi tiết kiến trúc, cấu trúc dữ liệu, các thuật toán xử lý layout ghế, cơ chế ghép đôi/ghép hàng, và luồng tương tác của **Trình chỉnh sửa sơ đồ ghế (Seat Editor)** trong trang quản trị BingeBox Cinema.

---

## 1. TỔNG QUAN VỀ SEAT EDITOR

Trình chỉnh sửa sơ đồ ghế là công cụ đồ họa trực quan (WYSIWYG - What You See Is What You Get) dành cho Quản trị viên (Admin) nhằm:
- Thiết kế ma trận ghế và lối đi linh hoạt cho từng phòng chiếu.
- Phân loại từng vị trí ghế: **Ghế Thường**, **Ghế VIP**, **Ghế Đôi (Couple / Sweetbox)**, hoặc **Ô Trống / Lối Đi (Blocked)**.
- Tự động đánh số thứ tự ghế và tên hàng theo chuẩn điện ảnh (`A1, A2, ..., B1, B2, ...`).
- Tự động ghép nối và đồng bộ mã ghế đối tác (`partnerSeatCode`) cho ghế đôi.
- Sao chép (clone) định dạng hàng ghế để dựng nhanh layout phòng chiếu lớn.
- Chuyển đổi dữ liệu ma trận 2D thành payload phẳng chuẩn hóa gửi về Backend để tính toán `seatLayout` và `totalSeats`.

---

## 2. SƠ ĐỒ KIẾN TRÚC & DÒNG DỮ LIỆU (DATA FLOW)

```
[ Giao diện Admin (/admin/seat) ]
  ├── 1. Component RoomInfo
  │      └── Chọn Rạp phim (Cinema) & Phòng chiếu (Room)
  │
  ├── 2. Component SeatList (Hiển thị Layout Ma trận)
  │      ├── Đọc state `rows` từ Zustand Store `useSeatStore`
  │      ├── Render danh sách hàng ghế (Row A, Row B, ...)
  │      ├── Thêm hàng bên dưới (addRowBelow) / Xóa hàng (removeRow)
  │      └── Button "Lưu sơ đồ" -> Gửi payload phẳng lên Backend API
  │
  ├── 3. Component SeatItem (Từng vị trí ghế / ô trống)
  │      ├── Render màu sắc theo `seatType.color`
  │      ├── Style liền khối cho ghế đôi (Couple)
  │      ├── Style viền nét đứt cho ô trống (Blocked / Lối đi)
  │      └── Click -> Cập nhật `selectedSeat: { rowIndex, seatIndex }`
  │
  └── 4. Component SeatEditor (Bảng điều khiển tác vụ ghế)
         ├── Đổi loại ghế (Standard, VIP, ...)
         ├── Chèn ghế mới (Trái / Phải)
         ├── Chèn ô trống lối đi (Trái / Phải)
         ├── Ghép thành ghế đôi / Hủy ghế đôi
         └── Xóa ghế
```

---

## 3. CẤU TRÚC STATE TẬP TRUNG (ZUSTAND STORE)

Toàn bộ trạng thái và logic thay đổi layout ghế được đóng gói trong Zustand store tại [useSeatStore.ts](file:///f:/Project/BINGEBOX/bingebox_fe/src/stores/useSeatStore.ts).

### 3.1. Data Interface

```typescript
interface SeatItem {
    id: string;               // ID tạm phía FE (crypto.randomUUID()) hoặc Mongo _id
    _id?: string;             // Mongo ID (nếu đã lưu trong DB)
    code: string;             // Mã ghế hiển thị (vd: "A1", "A2" hoặc "TRỐNG")
    row: string;              // Ký tự hàng ("A", "B", "C"...)
    column?: number;          // Vị trí cột (1-indexed)
    isBlocked: boolean;       // true nếu là ô trống/lối đi (không tính là ghế bán)
    isCoupleSeat: boolean;    // true nếu là ghế đôi
    partnerSeatCode?: string; // Mã ghế đối tác ghép đôi (vd: A1 ghép với A2)
    partnerSeat?: string;     // ID ghế đối tác (nếu có từ DB)
    seatType?: {              // Cấu hình loại ghế
        _id: string;
        name: string;
        color: string;
    };
    seatTypeId?: string;
}

interface RowItem {
    rowKey: string;           // Tên hàng: "A", "B", "C"...
    seats: SeatItem[];        // Danh sách ghế và ô trống trong hàng
}

interface SeatStore {
    rows: RowItem[];
    selectedSeat: { rowIndex: number; seatIndex: number } | null;
    
    // Actions
    setInitialRows: (seats: any[], categories: any[]) => void;
    initDefaultLayout: (defaultCategory: any) => void;
    setSelectedSeat: (selected: { rowIndex: number; seatIndex: number } | null) => void;
    changeSeatType: (categoryId: string, categories: any[]) => void;
    toggleCoupleSeat: (categories: any[]) => void;
    insertSeat: (isBlocked: boolean, side: "left" | "right") => void;
    removeSeat: () => void;
    addRowBelow: (rowIndex: number) => void;
    removeRow: (rowIndex: number) => void;
}
```

---

## 4. CHI TIẾT CÁC THUẬT TOÁN XỬ LÝ GHẾ CỐT LÕI

### 4.1. Thuật toán Reindex Toàn bộ Sơ đồ (`reindexAll`)

Khi thêm mới hoặc xóa một hàng ghế ở giữa sơ đồ, tên các hàng (`rowKey`) và mã ghế (`code`) của các hàng phía sau sẽ bị xáo trộn. Hàm `reindexAll` đảm bảo đồng bộ lại toàn bộ ma trận:

```typescript
const reindexAll = (allRows: any[]) => {
    return allRows.map((row, rIdx) => {
        // 1. Gán lại tên hàng theo bảng chữ cái ASCII: 0 -> 'A', 1 -> 'B', 2 -> 'C'...
        const newRowKey = String.fromCharCode(65 + rIdx);
        let seatCount = 1;

        // 2. Đánh lại số ghế: Chỉ tăng số đếm đối với ghế hợp lệ, ô trống đánh nhãn "TRỐNG"
        const newSeats = row.seats.map((s: any) => {
            const newCode = s.isBlocked ? "TRỐNG" : `${newRowKey}${seatCount++}`;
            return { ...s, row: newRowKey, code: newCode };
        });

        // 3. Tái liên kết partnerSeatCode cho các cặp ghế đôi
        const finalSeats = newSeats.map((s: any, sIdx: number) => {
            if (s.isCoupleSeat) {
                const next = newSeats[sIdx + 1];
                const prev = newSeats[sIdx - 1];
                if (next?.isCoupleSeat && next.partnerSeatCode !== s.code) {
                    return { ...s, partnerSeatCode: next.code };
                }
                if (prev?.isCoupleSeat) {
                    return { ...s, partnerSeatCode: prev.code };
                }
            }
            return s;
        });

        return { ...row, rowKey: newRowKey, seats: finalSeats };
    });
};
```

---

### 4.2. Thuật toán Đồng bộ Mã ghế Nội bộ Hàng (`syncRowData`)

Khi chèn thêm hoặc xóa một ghế/ô trống trong một hàng cụ thể, hàm `syncRowData` đánh lại số thứ tự từ 1 đến hết hàng đó mà không cần can thiệp tới các hàng khác:

```typescript
const syncRowData = (row: any) => {
    let count = 1;
    // 1. Đánh lại mã code ghế dựa vào rowKey hiện tại
    const updatedSeats = row.seats.map((s: any) => {
        const newCode = s.isBlocked ? "TRỐNG" : `${row.rowKey}${count++}`;
        return { ...s, code: newCode };
    });

    // 2. Cập nhật lại partnerSeatCode cho ghế đôi liền kề trước hoặc sau
    return updatedSeats.map((s: any, idx: number) => {
        if (s.isCoupleSeat) {
            const prev = updatedSeats[idx - 1];
            const next = updatedSeats[idx + 1];
            if (prev && prev.isCoupleSeat && (prev.partnerSeatCode === s.code || s.partnerSeatCode === prev.code)) {
                return { ...s, partnerSeatCode: prev.code };
            }
            if (next && next.isCoupleSeat) {
                return { ...s, partnerSeatCode: next.code };
            }
        }
        return s;
    });
};
```

---

### 4.3. Logic Ghép Đôi & Hủy Ghép Đôi Ghế (`toggleCoupleSeat`)

Quy tắc ghép đôi: Một ghế chỉ có thể ghép đôi với **ghế liền kề bên phải** của nó.

```
[ Ghế A1 (Single) ] + [ Ghế A2 (Single) ] 
                   |
                   v (Bấm "Ghép thành ghế đôi")
[ Ghế A1 (Couple - Partner: A2) ][ Ghế A2 (Couple - Partner: A1) ]
```

- **Điều kiện ghép đôi:**
  1. Ghế đang chọn phải không phải là ô trống (`!seat.isBlocked`).
  2. Phải tồn tại một ghế liền kề ngay bên phải (`newSeats[seatIndex + 1]`).
  3. Ghế bên phải không được là ô trống (`!nextSeat.isBlocked`) và chưa từng là ghế đôi (`!nextSeat.isCoupleSeat`).
  4. Cập nhật `isCoupleSeat = true`, liên kết chéo `partnerSeatCode`, và áp dụng `seatType` của danh mục Ghế đôi.
- **Hủy ghép đôi:**
  - Nếu ghế đang chọn đã là ghế đôi $\rightarrow$ Tìm ghế đối tác thông qua `partnerSeatCode`.
  - Đặt lại cả 2 ghế về `isCoupleSeat = false`, `partnerSeatCode = undefined`, chuyển loại ghế về danh mục mặc định (Ghế thường).

---

### 4.4. Logic Chèn Ghế & Ô Trống (`insertSeat`)

- **Tham số:** `isBlocked: boolean` (Ghế thật hay ô trống), `side: "left" | "right"` (Chèn bên trái hay bên phải).
- **Ràng buộc an toàn (Validation):**
  - Không cho phép chèn thêm vị trí vào **giữa một cặp ghế đôi đang ghép**.
  ```typescript
  if (side === "right" && seat.isCoupleSeat && nextSeat?.code === seat.partnerSeatCode) {
      toast.error("Không thể chèn vào giữa ghế đôi!");
      return;
  }
  if (side === "left" && seat.isCoupleSeat && prevSeat?.code === seat.partnerSeatCode) {
      toast.error("Không thể chèn vào giữa ghế đôi!");
      return;
  }
  ```
- **Thực hiện:**
  - Tạo đối tượng `newSeat` mới với ID ngẫu nhiên (`crypto.randomUUID()`).
  - Dùng `Array.prototype.splice()` để chèn vào mảng `seats`.
  - Chạy `syncRowData()` để đánh lại mã ghế trong hàng.

---

### 4.5. Logic Xóa Ghế (`removeSeat`)

- **Ràng buộc an toàn:**
  - Không cho phép xóa hết toàn bộ ghế trong hàng (Hàng phải có tối thiểu 1 vị trí).
- **Thực hiện:**
  - Nếu ghế đang xóa là **ghế đơn**: Xóa 1 phần tử tại `seatIndex`.
  - Nếu ghế đang xóa là **ghế đôi**: Tự động lọc và xóa đồng thời cả 2 ghế trong cặp (`s.code !== seat.code && s.code !== seat.partnerSeatCode`).
  - Chạy `syncRowData()` và hủy chọn ghế hiện tại (`selectedSeat: null`).

---

### 4.6. Logic Nhân bản & Xóa Hàng (`addRowBelow` & `removeRow`)

- **Thêm hàng bên dưới (`addRowBelow`):**
  - Sao chép toàn bộ cấu trúc (vị trí ghế, vị trí ô trống, loại ghế) từ hàng được chọn.
  - Reset trạng thái ghế đôi về đơn để tránh lỗi liên kết chéo giữa các hàng.
  - Chèn hàng mới vào vị trí `rowIndex + 1`.
  - Chạy `reindexAll()` để cập nhật lại toàn bộ chữ cái tên hàng và số ghế.
- **Xóa hàng (`removeRow`):**
  - Kiểm tra điều kiện: Phòng chiếu phải có ít nhất 1 hàng ghế (`rows.length > 1`).
  - Lọc bỏ hàng tại vị trí `rowIndex` và chạy `reindexAll()`.

---

## 5. CÁC COMPONENT GIAO DIỆN & TƯƠNG TÁC

### 5.1. `SeatPage` ([page.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/page.tsx))
- Điểm đầu vào chính (Root Entry).
- Quản lý state phòng chiếu đang chọn (`selectedRoom`).
- Render `RoomInfo` để chọn Rạp/Phòng; khi đã chọn phòng thì render `SeatList`.

### 5.2. `RoomInfo` ([room-info.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/room-info.tsx))
- Tích hợp 2 Combobox / Popover có hỗ trợ tìm kiếm Debounce (500ms):
  1. Chọn Rạp chiếu (`useCinemaList`).
  2. Chọn Phòng chiếu thuộc rạp đã chọn (`useRoomList`).
- Hiển thị bảng tóm tắt thông số phòng: Định dạng (2D/3D/IMAX), Tổng số hàng/cột hiện tại, Trạng thái hoạt động.

### 5.3. `SeatList` ([seat-list.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/seat-list.tsx))
- Nhận `roomId` từ props. Lấy dữ liệu ghế hiện có qua `useSeatListByRoom(roomId)` và nạp vào store qua `setInitialRows()`.
- Hiển thị nút bấm `Khởi tạo hàng ghế đầu tiên` nếu phòng chưa có ghế nào (`initDefaultLayout`).
- Hiển thị ma trận các hàng ghế kèm nút `+` (thêm hàng dưới) và `Trash2` (xóa hàng).
- Nút **"Lưu sơ đồ"**:
  - Chuyển đổi dữ liệu `rows` thành mảng phẳng `formattedSeats` với `column` tăng dần từ 1 đến hết hàng:
  ```typescript
  const formattedSeats = rows.flatMap((row) => {
      let currentColumn = 1;
      return row.seats.map((seat: any) => {
          const seatPayload: any = {
              code: seat.code,
              row: row.rowKey,
              column: currentColumn++,
              isBlocked: seat.isBlocked,
              seatTypeId: seat.seatType?._id || seat.seatTypeId,
          };
          if (seat.isCoupleSeat) {
              seatPayload.isCoupleSeat = true;
              seatPayload.partnerSeatCode = seat.partnerSeatCode;
          }
          if (seat.isBlocked) {
              seatPayload.seatTypeId = undefined;
          }
          return seatPayload;
      });
  });
  ```
  - Gọi mutation `updateSeats({ roomId, data: { seats: formattedSeats } })`.

### 5.4. `SeatItem` ([seat-item.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/seat-item.tsx))
- Hiển thị trạng thái đồ họa của từng ô:
  - **Ô trống (`isBlocked`)**: Viền nét đứt `border-dashed border-zinc-600`, dấu gạch ngang mờ `—`.
  - **Ghế thường**: Nền màu theo `seat.seatType.color`, bo góc `rounded-md`.
  - **Ghế đôi**: Loại bỏ bo góc ở giữa để nối liền khối `rounded-none first:rounded-l-lg last:rounded-r-lg border-x border-white/10`.
  - **Đang chọn (`active`)**: Scale phóng to 110%, viền trắng sáng `ring-2 ring-white z-10`.

### 5.5. `SeatEditor` ([seat-editor.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/seat-editor.tsx))
- Bảng công cụ điều khiển xuất hiện ở góc dưới khi quản trị viên nhấp chọn một ghế:
  - **Dropdown Loại ghế**: Chọn nhanh giữa Ghế thường, VIP... (Tự động lọc bỏ loại ghế đôi).
  - **Nhóm nút chèn**: `+ Ghế trái`, `+ Ghế phải`, `+ Trống trái`, `+ Trống phải`.
  - **Nút Ghép / Hủy ghế đôi**: Đổi màu động (Xanh khi có thể ghép, Đỏ khi đang là ghế đôi).
  - **Nút Xóa ghế**: Xóa ghế hoặc cặp ghế đôi ra khỏi layout.

---

## 6. DANH SÁCH FILE LIÊN QUAN TRONG PROJECT

| STT | File Path | Loại | Vai trò chính |
|---|---|---|---|
| 1 | [useSeatStore.ts](file:///f:/Project/BINGEBOX/bingebox_fe/src/stores/useSeatStore.ts) | Zustand Store | Trung tâm quản lý toàn bộ state `rows`, `selectedSeat` và các thuật toán `reindexAll`, `syncRowData`, chèn, xóa, ghép đôi. |
| 2 | [seat/page.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/page.tsx) | Next.js Page | Trang tổng quan quản lý ghế trong Admin dashboard. |
| 3 | [room-info.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/room-info.tsx) | Component | Bộ lọc chọn Rạp/Phòng chiếu kèm thông tin chi tiết phòng. |
| 4 | [seat-list.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/seat-list.tsx) | Component | Hiển thị sơ đồ ghế ma trận, các nút thêm/xóa hàng và nút lưu layout. |
| 5 | [seat-item.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/seat-item.tsx) | Component | Render giao diện từng ghế, ghế đôi và ô trống lối đi. |
| 6 | [seat-editor.tsx](file:///f:/Project/BINGEBOX/bingebox_fe/src/app/admin/seat/seat-editor.tsx) | Component | Toolbar thao tác điều khiển ghế (đổi loại, chèn, xóa, ghép đôi). |
| 7 | [useSeatQuery.ts](file:///f:/Project/BINGEBOX/bingebox_fe/src/queries/useSeatQuery.ts) | React Query | Hooks gọi API `useSeatListByRoom` (lấy sơ đồ) và `useUpdateSeatByRoom` (lưu sơ đồ). |
| 8 | [useSeatTypeQuery.ts](file:///f:/Project/BINGEBOX/bingebox_fe/src/queries/useSeatTypeQuery.ts) | React Query | Hook lấy danh sách cấu hình loại ghế (`useSeatTypeList`). |
| 9 | [seat.service.ts (BE)](file:///f:/Project/BINGEBOX/bingebox_be/src/modules/seat/seat.service.ts) | Backend Service | Xử lý `updateSeat`: Xóa ghế cũ, chèn ghế mới, link `partnerSeat` trong DB và tính lại `seatLayout` cho Room. |
