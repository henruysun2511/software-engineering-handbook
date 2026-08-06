## CÁC LỆNH GIT QUAN TRỌNG CHO BACKEND
 
Git là công cụ quản lý phiên bản mã nguồn không thể thiếu khi làm việc nhóm trong dự án Backend. Phần phụ lục này tổng hợp các lệnh Git thường dùng nhất, sắp xếp theo mục đích sử dụng thay vì liệt kê ngẫu nhiên, giúp dễ tra cứu khi cần.
 
### A.1. Khởi tạo & thiết lập
 
| Lệnh | Mô tả |
|---|---|
| `git init` | Khởi tạo một kho lưu trữ Git mới trong thư mục hiện tại. |
| `git clone <url>` | Sao chép toàn bộ một kho lưu trữ từ xa (ví dụ trên GitHub) về máy. |
| `git config --global user.name "..."` / `user.email "..."` | Thiết lập tên và email dùng để gắn vào mỗi commit, giúp phân biệt ai là người thực hiện thay đổi. |
 
### A.2. Theo dõi & lưu thay đổi (Staging & Commit)
 
| Lệnh | Mô tả |
|---|---|
| `git status` | Xem những tệp nào đã thay đổi, đã được thêm vào staging area hay chưa. Đây là lệnh nên gõ thường xuyên trước khi commit. |
| `git add <file>` / `git add .` | Đưa các thay đổi vào "staging area" — khu vực tạm chuẩn bị trước khi lưu chính thức thành một commit. |
| `git commit -m "message"` | Lưu lại các thay đổi đã staging thành một điểm mốc (commit) trong lịch sử dự án, kèm theo một dòng mô tả ngắn gọn về thay đổi đó. |
| `git diff` | So sánh chi tiết nội dung đã thay đổi nhưng chưa được commit, giúp kiểm tra lại trước khi lưu. |
 
### A.3. Quản lý nhánh (Branch)
 
Trong dự án Backend làm việc nhóm, mỗi tính năng hoặc bản sửa lỗi thường được phát triển trên một nhánh (branch) riêng, tách biệt khỏi nhánh chính (`main`/`master`), tránh ảnh hưởng đến mã nguồn đang chạy ổn định.
 
| Lệnh | Mô tả |
|---|---|
| `git branch` | Liệt kê các nhánh hiện có trong dự án. |
| `git checkout <branch>` hoặc `git switch <branch>` | Chuyển sang làm việc trên một nhánh khác. |
| `git checkout -b <branch>` hoặc `git switch -c <branch>` | Tạo một nhánh mới và chuyển sang nhánh đó ngay lập tức — thường dùng khi bắt đầu phát triển một tính năng mới. |
| `git merge <branch>` | Gộp các thay đổi từ một nhánh khác vào nhánh hiện tại, thường dùng khi một tính năng đã hoàn thành và cần đưa vào nhánh chính. |
| `git rebase <branch>` | Cũng nhằm mục đích gộp thay đổi giữa các nhánh, nhưng theo cách sắp xếp lại lịch sử commit sao cho "thẳng hàng", giúp lịch sử dự án gọn gàng, dễ theo dõi hơn so với `merge`. |
 
### A.4. Đồng bộ với kho lưu trữ từ xa (Remote)
 
| Lệnh | Mô tả |
|---|---|
| `git fetch` | Tải về các thay đổi mới nhất từ kho lưu trữ từ xa, nhưng **chưa** áp dụng vào nhánh đang làm việc — giúp xem trước những gì đã thay đổi. |
| `git pull` | Tương đương `git fetch` kết hợp `git merge` — tải về và gộp luôn các thay đổi mới nhất từ remote vào nhánh hiện tại. |
| `git push` | Đẩy các commit đã lưu ở máy cá nhân lên kho lưu trữ từ xa, để đồng đội khác có thể thấy được thay đổi. |
| `git push -u origin <branch>` | Đẩy một nhánh mới lên remote lần đầu tiên, đồng thời thiết lập liên kết mặc định để các lần `push`/`pull` sau không cần chỉ định lại tên nhánh. |
 
### A.5. Xem lại lịch sử thay đổi
 
| Lệnh | Mô tả |
|---|---|
| `git log` | Xem danh sách lịch sử các commit đã thực hiện, kèm tác giả, thời gian và nội dung mô tả. |
| `git show <commit>` | Xem chi tiết nội dung thay đổi của một commit cụ thể. |
| `git blame <file>` | Xem từng dòng trong một tệp được thay đổi lần cuối bởi commit nào, ai là người thực hiện — hữu ích khi cần truy tìm nguồn gốc của một đoạn code gây lỗi. |
 
### A.6. Hoàn tác thay đổi
 
Đây là nhóm lệnh quan trọng giúp xử lý khi commit hoặc thay đổi bị sai sót, cần đặc biệt cẩn thận khi sử dụng trên các nhánh dùng chung với cả nhóm.
 
| Lệnh | Mô tả |
|---|---|
| `git stash` | Tạm cất các thay đổi hiện tại chưa muốn commit sang một nơi lưu trữ tạm, giúp chuyển nhánh làm việc khác (ví dụ xử lý gấp một lỗi phát sinh) mà không mất thay đổi đang dang dở. Dùng `git stash pop` để lấy lại thay đổi đã cất. |
| `git restore <file>` (hoặc `git checkout -- <file>`) | Hủy bỏ các thay đổi chưa staging trên một tệp, đưa tệp về trạng thái giống với commit gần nhất. |
| `git reset` | Đưa nhánh hiện tại quay lại một commit trước đó, có thể giữ hoặc xóa luôn các thay đổi liên quan. **Chỉ nên dùng trên nhánh cá nhân**, vì lệnh này viết lại lịch sử, có thể gây xung đột nếu đồng đội đã tải về lịch sử cũ. |
| `git revert <commit>` | Tạo ra một commit **mới** có tác dụng đảo ngược nội dung của một commit trước đó, thay vì xóa commit cũ đi. Đây là cách an toàn hơn `reset` để hoàn tác thay đổi trên các nhánh dùng chung, vì không làm thay đổi lịch sử đã có. |
 
### A.7. Một số lệnh hữu ích khác
 
| Lệnh | Mô tả |
|---|---|
| `git cherry-pick <commit>` | Lấy riêng một commit cụ thể từ nhánh này áp dụng sang nhánh khác, thường dùng khi cần đưa gấp một bản sửa lỗi (hotfix) sang nhánh phát hành mà không muốn gộp toàn bộ nhánh. |
| `git tag <tên>` | Đánh dấu một điểm mốc quan trọng trong lịch sử dự án (thường dùng để đánh dấu các phiên bản phát hành, ví dụ `v1.0.0`). |
| `.gitignore` | Không phải một lệnh, mà là tệp cấu hình liệt kê những tệp/thư mục Git nên bỏ qua, không đưa vào quản lý phiên bản — ví dụ: tệp môi trường chứa thông tin nhạy cảm (`.env`), thư mục cài đặt thư viện (`node_modules`). |
 
