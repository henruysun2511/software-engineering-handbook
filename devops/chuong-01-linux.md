# Chương 1: Linux

> **Mức độ quan trọng:** ⭐⭐⭐⭐⭐  
> **Đối tượng:** Sinh viên CNTT, Backend Developer (Node.js/NestJS), trình độ Intern → Junior  
> **Mục tiêu chương:** Nắm vững nền tảng Linux để vận hành, debug và deploy ứng dụng backend trên môi trường server thực tế.

---

## 1.1. Linux Là Gì?

### 1.1.1. Khái niệm

**Linux** là một hệ điều hành mã nguồn mở (open-source operating system) được xây dựng dựa trên nhân Linux (Linux kernel), do Linus Torvalds tạo ra vào năm 1991. Linux không phải là một hệ điều hành duy nhất mà là một "họ" hệ điều hành — mỗi phiên bản được gọi là một **distribution (distro)**, ví dụ: Ubuntu, Debian, CentOS, Alpine.

Linux không chỉ là một lựa chọn — trong thực tế, **hơn 90% server trên thế giới chạy Linux**. Điều đó có nghĩa là: dù bạn là backend developer viết code trên Windows hay macOS, ứng dụng của bạn rốt cuộc cũng sẽ chạy trên một máy chủ Linux.

### 1.1.2. Vì sao Linux thống trị thế giới server?

| Lý do | Giải thích |
|---|---|
| **Miễn phí & mã nguồn mở** | Không tốn chi phí bản quyền, có thể tùy biến sâu |
| **Ổn định & hiệu năng cao** | Chạy hàng năm không cần khởi động lại |
| **Bảo mật tốt** | Hệ thống phân quyền chặt chẽ |
| **Tiêu thụ tài nguyên thấp** | Chạy tốt trên server không có giao diện đồ họa |
| **Cộng đồng lớn** | Mọi vấn đề đều có giải pháp được ghi chép |

### 1.1.3. Linux giải quyết vấn đề gì cho Backend Developer?

Với tư cách là backend developer, bạn sẽ gặp Linux trong các tình huống sau:

- **SSH vào VPS** để kiểm tra ứng dụng đang chạy.
- **Đọc log** khi ứng dụng bị lỗi trên production.
- **Cài đặt và quản lý** các công cụ như Docker, Nginx.
- **Viết script** tự động hóa tác vụ deploy.
- **Debug** khi container bị crash hoặc hết tài nguyên.

---

## 1.2. Linux File System

### 1.2.1. Khái niệm

Linux tổ chức toàn bộ dữ liệu theo một **cây thư mục đơn** (single directory tree) bắt đầu từ thư mục gốc `/` (root). Khác với Windows có nhiều ổ đĩa (C:\, D:\), trong Linux mọi thứ đều nằm dưới `/`.

### 1.2.2. Cấu trúc thư mục chuẩn

```
/
├── bin/        # Các lệnh cơ bản (ls, cp, mv...)
├── etc/        # File cấu hình hệ thống (nginx.conf, hosts...)
├── home/       # Thư mục cá nhân của từng user (/home/alice, /home/bob)
├── root/       # Thư mục cá nhân của user root
├── var/        # Dữ liệu thay đổi thường xuyên (log, cache, db...)
│   └── log/    # Log hệ thống (/var/log/nginx/, /var/log/syslog)
├── tmp/        # File tạm thời, tự xóa khi reboot
├── usr/        # Phần mềm do người dùng cài đặt
│   └── local/  # Phần mềm cài thủ công
├── opt/        # Phần mềm bên thứ ba (không qua package manager)
├── proc/       # Thông tin về process đang chạy (virtual filesystem)
├── dev/        # Thiết bị phần cứng (disk, terminal...)
├── mnt/        # Mount point tạm thời cho ổ đĩa ngoài
└── srv/        # Dữ liệu của các service (web, ftp...)
```

### 1.2.3. Những thư mục quan trọng nhất với Backend Developer

| Thư mục | Vai trò thực tế |
|---|---|
| `/etc/nginx/` | Cấu hình Nginx reverse proxy |
| `/var/log/` | Đọc log khi debug production |
| `/home/<user>/` | Thư mục làm việc chính trên server |
| `/tmp/` | File tạm, upload trước khi xử lý |
| `/opt/` | Đặt source code ứng dụng tại đây |

### 1.2.4. Đường dẫn tuyệt đối vs tương đối

```bash
# Đường dẫn tuyệt đối — bắt đầu từ /
/home/alice/project/src/main.ts

# Đường dẫn tương đối — bắt đầu từ thư mục hiện tại
./src/main.ts      # Thư mục con src trong thư mục hiện tại
../config.yaml     # Lên một cấp rồi tìm file config.yaml
```

---

## 1.3. Command Line

### 1.3.1. Terminal, Shell và Command Line

- **Terminal** (hay Terminal Emulator): Cửa sổ giao diện để bạn gõ lệnh.
- **Shell**: Chương trình phiên dịch lệnh bạn nhập. Shell phổ biến nhất là **Bash** (Bourne Again Shell).
- **Command Line** (CLI): Giao diện dòng lệnh — đối lập với giao diện đồ họa (GUI).

### 1.3.2. Cấu trúc một lệnh Linux

```
command  [options]  [arguments]
   ↑         ↑           ↑
 Lệnh    Tùy chọn    Đối số/Tham số
```

**Ví dụ:**

```bash
ls    -la    /home/alice
 ↑     ↑          ↑
 ls  (list)  long format  thư mục cần liệt kê
```

### 1.3.3. Ký hiệu Prompt

```bash
alice@ubuntu:~$         # User thường
root@ubuntu:~#          # User root (superuser)

# alice    = tên user
# ubuntu   = hostname máy chủ
# ~        = thư mục hiện tại (~ là viết tắt của /home/alice)
# $        = dấu hiệu user thường
# #        = dấu hiệu root
```

### 1.3.4. Các phím tắt cơ bản trên Terminal

| Phím tắt | Chức năng |
|---|---|
| `Tab` | Tự động hoàn thành lệnh hoặc đường dẫn |
| `↑ / ↓` | Duyệt lại lệnh đã gõ |
| `Ctrl + C` | Dừng tiến trình đang chạy |
| `Ctrl + L` | Xóa màn hình (tương đương lệnh `clear`) |
| `Ctrl + A` | Di chuyển con trỏ về đầu dòng |
| `Ctrl + E` | Di chuyển con trỏ về cuối dòng |

### 1.3.5. Lấy trợ giúp

```bash
man ls           # Xem manual của lệnh ls (gõ q để thoát)
ls --help        # Xem tóm tắt các option
which node       # Xem lệnh node đang nằm ở đâu
```

---

## 1.4. File & Directory

### 1.4.1. `pwd` — Print Working Directory

Hiển thị đường dẫn tuyệt đối của thư mục đang làm việc.

```bash
pwd
# Output: /home/alice/projects/my-api
```

### 1.4.2. `ls` — List Directory Contents

```bash
ls                  # Liệt kê file trong thư mục hiện tại
ls /etc/nginx       # Liệt kê file trong /etc/nginx
ls -l               # Hiển thị chi tiết (quyền, kích thước, ngày sửa)
ls -a               # Hiển thị cả file ẩn (bắt đầu bằng dấu .)
ls -la              # Kết hợp: chi tiết + file ẩn
ls -lh              # Chi tiết + kích thước dễ đọc (KB, MB, GB)
ls -lt              # Sắp xếp theo thời gian sửa đổi (mới nhất trước)
```

**Giải thích output của `ls -la`:**

```
drwxr-xr-x  3 alice alice 4096 Jul 10 14:30 src
-rw-r--r--  1 alice alice  892 Jul 10 14:28 package.json
-rwxr-xr-x  1 alice alice 1240 Jul 10 14:25 start.sh

# Cột 1: Loại file + Quyền (d = directory, - = file thường)
# Cột 2: Số hard link
# Cột 3: Owner (chủ sở hữu)
# Cột 4: Group
# Cột 5: Kích thước (bytes)
# Cột 6-7: Ngày giờ sửa đổi
# Cột 8: Tên file
```

### 1.4.3. `cd` — Change Directory

```bash
cd /var/log          # Di chuyển đến đường dẫn tuyệt đối
cd projects/my-api   # Di chuyển theo đường dẫn tương đối
cd ~                 # Về thư mục home của user hiện tại
cd ..                # Lên một cấp thư mục cha
cd -                 # Quay lại thư mục vừa rời
```

### 1.4.4. `mkdir` — Make Directory

```bash
mkdir logs                        # Tạo thư mục logs trong thư mục hiện tại
mkdir -p /opt/app/logs/2024       # Tạo cả chuỗi thư mục lồng nhau
mkdir -p src/{controllers,services,modules}  # Tạo nhiều thư mục con cùng lúc
```

### 1.4.5. `touch` — Tạo File Rỗng / Cập Nhật Timestamp

```bash
touch .env                    # Tạo file .env rỗng
touch README.md               # Tạo file README.md
touch file1.txt file2.txt     # Tạo nhiều file cùng lúc
```

### 1.4.6. `cp` — Copy

```bash
cp app.js app.js.bak              # Sao chép file (tạo bản backup)
cp -r src/ src-backup/            # Sao chép toàn bộ thư mục (r = recursive)
cp .env.example .env              # Dùng phổ biến: tạo .env từ template
cp -rp /opt/app /opt/app-old      # Sao chép và giữ nguyên permissions
```

### 1.4.7. `mv` — Move / Rename

```bash
mv old-name.txt new-name.txt      # Đổi tên file
mv app.log /var/log/app.log       # Di chuyển file sang thư mục khác
mv /tmp/release /opt/app          # Di chuyển thư mục
```

### 1.4.8. `rm` — Remove

> ⚠️ **Cảnh báo:** Linux không có Recycle Bin. File bị xóa bằng `rm` sẽ **mất vĩnh viễn**.

```bash
rm file.txt                   # Xóa file
rm -f file.txt                # Xóa file, không hỏi lại
rm -r node_modules/           # Xóa thư mục và toàn bộ nội dung bên trong
rm -rf dist/                  # Xóa mạnh — không hỏi, không báo lỗi
rm -i file.txt                # Xóa có hỏi xác nhận (an toàn hơn)

# TUYỆT ĐỐI KHÔNG BAO GIỜ chạy:
# rm -rf /          ← xóa toàn bộ hệ thống
# rm -rf /*         ← tương tự, chỉ khác cách viết
```

### 1.4.9. Xem nội dung file

```bash
cat package.json              # In toàn bộ nội dung file ra màn hình
cat -n main.ts                # In kèm số dòng

less app.log                  # Xem file lớn theo trang (q để thoát)
                              # Hỗ trợ tìm kiếm: gõ /keyword rồi Enter

head -n 20 app.log            # Xem 20 dòng đầu
tail -n 50 app.log            # Xem 50 dòng cuối
tail -f app.log               # Xem log realtime (f = follow, Ctrl+C để dừng)
tail -f -n 100 app.log        # Xem 100 dòng cuối rồi follow realtime
```

---

## 1.5. File Permission (Phân Quyền File)

### 1.5.1. Khái niệm

Linux sử dụng hệ thống phân quyền để kiểm soát **ai được làm gì** với file. Đây là nền tảng bảo mật của Linux.

### 1.5.2. Ba loại đối tượng

| Ký hiệu | Đối tượng | Mô tả |
|---|---|---|
| `u` | User (owner) | Người tạo/sở hữu file |
| `g` | Group | Nhóm user được gán quyền |
| `o` | Others | Tất cả user còn lại |

### 1.5.3. Ba loại quyền

| Ký hiệu | Số | Quyền | Với File | Với Thư mục |
|---|---|---|---|---|
| `r` | 4 | Read | Đọc nội dung | Liệt kê file bên trong |
| `w` | 2 | Write | Sửa nội dung | Tạo/xóa file bên trong |
| `x` | 1 | Execute | Chạy file như chương trình | Truy cập vào thư mục |

### 1.5.4. Đọc quyền file

```
-rwxr-xr--
 ↑↑↑↑↑↑↑↑↑
 │└┤└┤└┤└┘
 │ │ │ └── Others: r-- (chỉ đọc)
 │ │ └──── Group:  r-x (đọc + chạy)
 │ └────── User:   rwx (đọc + ghi + chạy)
 └──────── Loại: - (file thường), d (directory), l (symlink)
```

**Quy đổi sang số:**

```
rwx = 4+2+1 = 7
r-x = 4+0+1 = 5
r-- = 4+0+0 = 4

-rwxr-xr-- = 754
```

### 1.5.5. `chmod` — Change Mode

```bash
# Dạng ký hiệu
chmod u+x script.sh          # Thêm quyền execute cho owner
chmod g-w file.txt           # Bỏ quyền write của group
chmod o+r file.txt           # Thêm quyền read cho others
chmod a+x script.sh          # Thêm execute cho tất cả (a = all)
chmod ug+rw file.txt         # Thêm read+write cho user và group

# Dạng số (phổ biến hơn)
chmod 755 script.sh          # rwxr-xr-x (script chạy được, mọi người đọc được)
chmod 644 .env               # rw-r--r-- (chỉ owner đọc/ghi, others chỉ đọc)
chmod 600 id_rsa             # rw------- (chỉ owner đọc/ghi, SSH key yêu cầu)
chmod 777 upload/            # rwxrwxrwx (mọi người full quyền — dùng cẩn thận)

# Đệ quy (toàn bộ thư mục)
chmod -R 755 /opt/app        # Áp dụng cho tất cả file và thư mục con
```

**Quyền phổ biến trong thực tế:**

| Permission | Dùng khi nào |
|---|---|
| `755` | Script, thư mục web, binary |
| `644` | File cấu hình, source code |
| `600` | SSH private key, file chứa secret |
| `400` | File chỉ đọc, không ai được sửa |

### 1.5.6. `chown` — Change Owner

```bash
chown alice file.txt              # Đổi owner thành alice
chown alice:developers file.txt   # Đổi owner + group
chown -R alice:alice /opt/app     # Đổi đệ quy toàn bộ thư mục
chown root:root /etc/nginx/nginx.conf  # Thường dùng khi cấu hình hệ thống
```

### 1.5.7. `sudo` — Superuser Do

```bash
sudo apt update               # Chạy lệnh với quyền root
sudo chmod 644 /etc/hosts     # Sửa file cần quyền root
sudo -i                       # Chuyển sang shell root (dùng cẩn thận)
sudo su - alice               # Chuyển sang user alice
```

> 💡 **Thực tế:** Trên các server cloud (AWS, GCP, DigitalOcean), bạn thường login bằng user `ubuntu` hoặc `ec2-user`, rồi dùng `sudo` cho các tác vụ quản trị.

---

## 1.6. Process (Tiến Trình)

### 1.6.1. Khái niệm

**Process** là một chương trình đang được thực thi. Khi bạn chạy `node dist/main.js`, Linux tạo ra một process với ID duy nhất gọi là **PID** (Process ID).

### 1.6.2. `ps` — Process Status

```bash
ps                        # Liệt kê process của user hiện tại trong terminal này
ps aux                    # Liệt kê tất cả process của mọi user
ps aux | grep node        # Tìm process liên quan đến node
ps aux | grep -v grep     # Loại bỏ dòng grep khỏi kết quả
```

**Giải thích output của `ps aux`:**

```
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
alice     1234  2.1  1.5 123456 31456 ?        Ssl  10:30   0:05 node dist/main.js
nginx      567  0.0  0.1  12345  2345 ?        Ss   10:00   0:00 nginx: master

# USER: Tên user đang chạy process
# PID:  Process ID (dùng để kill)
# %CPU: % CPU đang dùng
# %MEM: % RAM đang dùng
# STAT: Trạng thái (S=sleeping, R=running, Z=zombie, D=disk wait)
# COMMAND: Lệnh khởi chạy
```

### 1.6.3. `top` và `htop` — Giám Sát Realtime

```bash
top         # Xem process realtime (nhấn q để thoát)
htop        # Phiên bản nâng cao, giao diện đẹp hơn (cần cài thêm)
```

**Các phím trong `top`:**
- `q` — Thoát
- `k` — Kill một process (nhập PID)
- `M` — Sắp xếp theo RAM
- `P` — Sắp xếp theo CPU
- `1` — Hiển thị từng CPU core

### 1.6.4. `kill` — Dừng Process

```bash
kill 1234                 # Gửi tín hiệu SIGTERM (dừng gracefully)
kill -9 1234              # Gửi tín hiệu SIGKILL (dừng ngay lập tức, không cleanup)
kill -15 1234             # Tương đương kill (SIGTERM)
killall node              # Kill tất cả process tên "node"
pkill -f "dist/main.js"   # Kill theo pattern tên lệnh
```

**Thứ tự nên dùng:**

```
kill PID       →  Chờ 5-10 giây  →  kill -9 PID
  (lịch sự)                          (cưỡng bức)
```

### 1.6.5. Chạy Process trong Background

```bash
node dist/main.js &          # Chạy background, thoát khi terminal đóng
nohup node dist/main.js &    # Chạy background, tiếp tục khi terminal đóng
jobs                         # Liệt kê background jobs
fg 1                         # Đưa job số 1 về foreground
bg 1                         # Đẩy job số 1 vào background
```

> 💡 **Thực tế:** Trong production, bạn sẽ dùng `systemd`, `pm2`, hoặc `Docker` thay vì `nohup` để quản lý process chuyên nghiệp hơn.

### 1.6.6. `systemctl` — Quản Lý Service

```bash
sudo systemctl start nginx          # Khởi động service
sudo systemctl stop nginx           # Dừng service
sudo systemctl restart nginx        # Khởi động lại
sudo systemctl reload nginx         # Tải lại cấu hình (không restart)
sudo systemctl status nginx         # Xem trạng thái + log gần nhất
sudo systemctl enable nginx         # Tự động chạy khi server boot
sudo systemctl disable nginx        # Không tự động chạy khi boot
sudo systemctl list-units --type=service  # Liệt kê tất cả service
```

---

## 1.7. Network

### 1.7.1. `ping` — Kiểm Tra Kết Nối

```bash
ping google.com              # Kiểm tra có kết nối internet không
ping 192.168.1.1             # Kiểm tra kết nối đến server nội bộ
ping -c 4 google.com         # Chỉ gửi 4 gói (c = count), tự thoát
ping -i 0.5 google.com       # Ping mỗi 0.5 giây
```

### 1.7.2. `curl` — Transfer Data

`curl` là công cụ cực kỳ mạnh để gửi HTTP request từ terminal.

```bash
# GET request cơ bản
curl http://localhost:3000/api/health

# GET với header
curl -H "Authorization: Bearer token123" http://localhost:3000/api/me

# POST với JSON body
curl -X POST http://localhost:3000/api/users \
  -H "Content-Type: application/json" \
  -d '{"name": "Alice", "email": "alice@example.com"}'

# Xem response header
curl -I http://localhost:3000      # Chỉ xem header
curl -v http://localhost:3000      # Verbose: xem cả request + response header

# Lưu response ra file
curl -o response.json http://api.example.com/data

# Theo dõi redirect
curl -L http://example.com

# Kiểm tra HTTPS với certificate tự ký (môi trường dev)
curl -k https://localhost:443/api/health

# Upload file
curl -X POST http://localhost:3000/api/upload \
  -F "file=@/path/to/file.pdf"
```

### 1.7.3. `wget` — Download File

```bash
wget https://example.com/file.zip              # Download file
wget -O output.zip https://example.com/file.zip  # Download và đặt tên
wget -q https://example.com/file.zip           # Quiet mode (ít output)
wget -c https://example.com/largefile.zip      # Tiếp tục download bị gián đoạn
wget -P /tmp/ https://example.com/script.sh    # Download vào thư mục /tmp/
```

### 1.7.4. `ss` và `netstat` — Xem Port Đang Mở

```bash
# ss (socket statistics — công cụ hiện đại hơn netstat)
ss -tlnp                   # Xem TCP port đang lắng nghe
ss -tlnp | grep :3000      # Kiểm tra port 3000 có đang mở không
ss -anp                    # Xem tất cả connection

# netstat (cần cài: sudo apt install net-tools)
netstat -tlnp              # Tương tự ss -tlnp
netstat -an | grep LISTEN  # Xem tất cả port đang lắng nghe
```

**Giải thích output:**

```
Netid  State    Recv-Q Send-Q  Local Address:Port   Peer Address:Port  Process
tcp    LISTEN   0      128     0.0.0.0:3000          0.0.0.0:*          users:(("node",pid=1234))
tcp    LISTEN   0      511     0.0.0.0:80            0.0.0.0:*          users:(("nginx",pid=567))

# 0.0.0.0:3000 → Lắng nghe trên tất cả interface, port 3000
# 127.0.0.1:5432 → Chỉ lắng nghe trên localhost (PostgreSQL)
```

### 1.7.5. `ssh` — Secure Shell

```bash
# Kết nối đến server
ssh alice@192.168.1.10               # Đăng nhập bằng password
ssh -i ~/.ssh/id_rsa alice@server.com  # Đăng nhập bằng SSH key
ssh -p 2222 alice@server.com         # Kết nối qua port tùy chỉnh

# Chạy lệnh từ xa mà không cần vào shell
ssh alice@server.com "ls /opt/app"
ssh alice@server.com "docker ps"

# Tạo SSH key pair
ssh-keygen -t ed25519 -C "your_email@example.com"
# Kết quả: ~/.ssh/id_ed25519 (private key) và ~/.ssh/id_ed25519.pub (public key)

# Copy public key lên server
ssh-copy-id alice@server.com
# Hoặc thủ công:
cat ~/.ssh/id_ed25519.pub >> ~/.ssh/authorized_keys  # Chạy trên server
```

### 1.7.6. `scp` — Secure Copy

```bash
# Upload file từ local lên server
scp file.txt alice@server.com:/home/alice/

# Upload cả thư mục
scp -r ./dist alice@server.com:/opt/app/

# Download file từ server về local
scp alice@server.com:/var/log/app.log ./logs/

# Copy kèm SSH key
scp -i ~/.ssh/id_rsa file.txt alice@server.com:/tmp/
```

---

## 1.8. Package Manager — `apt`

### 1.8.1. Khái niệm

**APT** (Advanced Package Tool) là trình quản lý gói của các distro Debian/Ubuntu. Nó giúp cài đặt, cập nhật và gỡ phần mềm một cách tự động — bao gồm cả việc xử lý các dependency.

### 1.8.2. Các lệnh cơ bản

```bash
# Cập nhật danh sách gói (luôn chạy trước khi cài mới)
sudo apt update

# Nâng cấp tất cả gói đã cài
sudo apt upgrade -y

# Cài đặt phần mềm
sudo apt install nginx -y
sudo apt install git curl wget vim htop -y   # Cài nhiều gói cùng lúc

# Gỡ cài đặt
sudo apt remove nginx                  # Xóa phần mềm (giữ lại cấu hình)
sudo apt purge nginx                   # Xóa phần mềm + cấu hình
sudo apt autoremove                    # Xóa các gói không còn cần thiết

# Tìm kiếm gói
apt search nodejs
apt show nginx                         # Xem thông tin chi tiết về gói

# Kiểm tra phần mềm đã cài
dpkg -l | grep nginx
which nginx                            # Tìm đường dẫn binary
nginx -v                               # Kiểm tra version
```

### 1.8.3. Cài Node.js trên Ubuntu (Ví dụ thực tế)

```bash
# Cách 1: Dùng NodeSource (khuyến nghị cho production)
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs

# Kiểm tra sau khi cài
node -v    # v20.x.x
npm -v     # 10.x.x

# Cách 2: Dùng nvm (khuyến nghị cho development)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc
nvm install 20
nvm use 20
nvm alias default 20
```

---

## 1.9. Environment Variable (Biến Môi Trường)

### 1.9.1. Khái niệm

**Environment Variable** là biến được lưu trong môi trường của shell/process, dùng để truyền cấu hình vào ứng dụng mà không cần hard-code trong source code. Đây là phương pháp chuẩn theo [The Twelve-Factor App methodology](https://12factor.net/config).

### 1.9.2. Tại sao cần Environment Variable?

Xem xét ví dụ sau trong NestJS:

```typescript
// Cách sai — hard-code thông tin nhạy cảm
const client = new Pool({
  host: "db.production.example.com",
  password: "super-secret-password",
  database: "myapp_prod",
});
```

Vấn đề:
- Password bị lộ nếu code được đẩy lên GitHub.
- Phải sửa code khi đổi môi trường (dev → staging → production).
- Không thể cấu hình khác nhau cho từng môi trường.

**Giải pháp:**

```typescript
// Cách đúng — dùng environment variable
const client = new Pool({
  host: process.env.DB_HOST,
  password: process.env.DB_PASSWORD,
  database: process.env.DB_NAME,
});
```

### 1.9.3. Làm việc với Environment Variable trên Linux

```bash
# Xem tất cả biến môi trường
env
printenv

# Xem một biến cụ thể
echo $PATH
echo $HOME
echo $USER
printenv NODE_ENV

# Đặt biến tạm thời (chỉ trong session hiện tại)
export NODE_ENV=production
export PORT=3000
export DB_HOST=localhost

# Đặt biến chỉ cho một lệnh
NODE_ENV=production node dist/main.js

# Xóa biến
unset NODE_ENV
```

### 1.9.4. Biến Môi Trường Hệ Thống Quan Trọng

```bash
echo $PATH     # Danh sách thư mục chứa các lệnh thực thi
echo $HOME     # Thư mục home của user hiện tại
echo $USER     # Tên user hiện tại
echo $PWD      # Thư mục làm việc hiện tại (giống pwd)
echo $SHELL    # Shell đang dùng (/bin/bash)
```

### 1.9.5. File `.bashrc` và `.bash_profile`

Để biến môi trường tồn tại vĩnh viễn qua các session:

```bash
# Chỉnh sửa .bashrc (cho interactive shell)
vim ~/.bashrc

# Thêm vào cuối file:
export NODE_ENV=production
export PORT=3000
export NVM_DIR="$HOME/.nvm"

# Áp dụng ngay mà không cần đăng xuất
source ~/.bashrc
```

### 1.9.6. File `.env` Trong Dự Án

```bash
# File .env (KHÔNG commit lên Git)
NODE_ENV=development
PORT=3000
DB_HOST=localhost
DB_PORT=5432
DB_NAME=myapp_dev
DB_USER=postgres
DB_PASSWORD=dev_password_123
JWT_SECRET=dev-jwt-secret-key
REDIS_URL=redis://localhost:6379
```

```bash
# File .env.example (COMMIT lên Git — template không có giá trị thật)
NODE_ENV=
PORT=3000
DB_HOST=
DB_PORT=5432
DB_NAME=
DB_USER=
DB_PASSWORD=
JWT_SECRET=
REDIS_URL=
```

```bash
# .gitignore — đảm bảo .env không bị commit
.env
.env.local
.env.production
```

---

## 1.10. Xem và Phân Tích Log

### 1.10.1. Các lệnh xem log

```bash
# Xem toàn bộ file log
cat /var/log/nginx/access.log

# Xem realtime (phổ biến nhất khi debug production)
tail -f /var/log/nginx/error.log
tail -f -n 100 /var/log/app/app.log

# Xem log hệ thống (systemd)
journalctl -u nginx                  # Log của service nginx
journalctl -u nginx -f               # Follow realtime
journalctl -u nginx --since "1 hour ago"  # Log trong 1 giờ qua
journalctl -n 50                     # 50 dòng log cuối
```

### 1.10.2. `grep` — Tìm Kiếm Trong File/Output

```bash
# Tìm kiếm cơ bản
grep "ERROR" app.log              # Tìm dòng chứa "ERROR"
grep -i "error" app.log           # Không phân biệt hoa thường
grep -n "ERROR" app.log           # Hiển thị số dòng
grep -c "ERROR" app.log           # Đếm số dòng khớp

# Kết hợp với tail (pattern thực dụng nhất khi debug)
tail -f app.log | grep "ERROR"    # Chỉ hiển thị dòng error realtime

# Tìm nhiều pattern
grep -E "ERROR|WARN|FATAL" app.log

# Tìm ngược (loại trừ)
grep -v "DEBUG" app.log           # Hiển thị tất cả trừ dòng DEBUG

# Tìm đệ quy trong nhiều file
grep -r "DB_PASSWORD" .           # Tìm trong tất cả file — kiểm tra secret lộ không
grep -r "localhost" /etc/nginx/   # Tìm trong cấu hình nginx
```

### 1.10.3. `find` — Tìm File và Thư Mục

```bash
find . -name "*.log"              # Tìm tất cả file .log trong thư mục hiện tại
find / -name "nginx.conf"         # Tìm nginx.conf trên toàn hệ thống
find . -name "*.env"              # Tìm file env (kiểm tra file nhạy cảm)
find . -type f -name "*.ts"       # Chỉ tìm file (không phải thư mục)
find . -type d -name "node_modules"  # Chỉ tìm thư mục

# Tìm file theo thời gian
find /var/log -mtime -1           # File sửa đổi trong 24h qua
find /tmp -mtime +7               # File cũ hơn 7 ngày

# Tìm và xóa (cẩn thận!)
find . -name "*.log" -delete      # Xóa tất cả file .log tìm được
find /tmp -mtime +7 -delete       # Dọn file tạm cũ hơn 7 ngày
```

---

## 1.11. Kiểm Tra Tài Nguyên Hệ Thống

### 1.11.1. `df` — Disk Free (Kiểm tra dung lượng ổ đĩa)

```bash
df -h                    # Xem dung lượng tất cả ổ đĩa (h = human readable)
df -h /                  # Xem dung lượng ổ đĩa root
df -h /opt               # Xem dung lượng ổ đĩa chứa thư mục /opt
```

**Output:**

```
Filesystem      Size  Used Avail Use% Mounted on
/dev/vda1        50G   32G   18G  65% /
tmpfs           1.9G     0  1.9G   0% /dev/shm

# 65% đã dùng — OK
# 90%+ thì cần dọn dẹp ngay
```

### 1.11.2. `du` — Disk Usage (Xem thư mục nào chiếm nhiều chỗ)

```bash
du -sh *                 # Xem kích thước từng thư mục trong thư mục hiện tại
du -sh /var/log/*        # Xem kích thước các thư mục log
du -sh node_modules/     # Kiểm tra kích thước node_modules
du -sh /var/lib/docker/  # Docker chiếm bao nhiêu dung lượng
```

### 1.11.3. `free` — Kiểm tra RAM

```bash
free -h                  # Xem RAM đang dùng bao nhiêu
```

**Output:**

```
               total        used        free      shared  buff/cache   available
Mem:           3.8Gi       1.2Gi       800Mi       45Mi       1.8Gi       2.3Gi
Swap:          2.0Gi       100Mi       1.9Gi

# available = RAM thực sự có thể dùng (2.3Gi)
# Nếu available thấp + swap đang dùng nhiều → ứng dụng thiếu RAM
```

---

## 1.12. `cron` — Lên Lịch Tác Vụ Tự Động

### 1.12.1. Khái niệm

**Cron** là công cụ lên lịch chạy lệnh hoặc script theo thời gian định kỳ. Phổ biến dùng cho: backup database, dọn log cũ, gửi email định kỳ.

### 1.12.2. Cú pháp crontab

```
┌───────────── Phút (0 - 59)
│ ┌───────────── Giờ (0 - 23)
│ │ ┌───────────── Ngày trong tháng (1 - 31)
│ │ │ ┌───────────── Tháng (1 - 12)
│ │ │ │ ┌───────────── Ngày trong tuần (0 - 6, 0 = Chủ nhật)
│ │ │ │ │
* * * * * lệnh_cần_chạy
```

```bash
# Chỉnh sửa crontab của user hiện tại
crontab -e

# Xem crontab hiện tại
crontab -l

# Xóa toàn bộ crontab
crontab -r
```

### 1.12.3. Ví dụ crontab thực tế

```bash
# Chạy script backup database mỗi ngày lúc 2:00 AM
0 2 * * * /opt/scripts/backup-db.sh >> /var/log/backup.log 2>&1

# Restart ứng dụng mỗi Chủ nhật lúc 3:00 AM
0 3 * * 0 systemctl restart myapp

# Xóa log cũ hơn 30 ngày, chạy mỗi ngày lúc 4:00 AM
0 4 * * * find /var/log/app -mtime +30 -delete

# Chạy mỗi 15 phút
*/15 * * * * /opt/scripts/health-check.sh

# Chạy lúc 9:00 AM mỗi thứ Hai đến thứ Sáu
0 9 * * 1-5 /opt/scripts/send-report.sh
```

---

## 1.13. Vim — Trình Soạn Thảo Trên Terminal

### 1.13.1. Tại sao cần biết Vim?

Trên server production, thường không có GUI. Bạn cần chỉnh sửa file cấu hình trực tiếp trong terminal. **Vim** (hoặc **Nano**) là công cụ không thể tránh.

### 1.13.2. Vim cơ bản — đủ dùng trên server

```bash
vim /etc/nginx/nginx.conf    # Mở file bằng vim
```

**Ba mode của Vim:**

```
Normal Mode (mặc định khi mở)
  → Dùng để: di chuyển, copy, xóa, tìm kiếm
  → Nhấn Esc để quay về mode này từ bất kỳ đâu

Insert Mode (gõ nội dung)
  → Vào mode này bằng: i (trước con trỏ), a (sau con trỏ), o (dòng mới)

Command Mode (lệnh đặc biệt)
  → Vào bằng: : (từ Normal mode)
```

**Các lệnh hay dùng:**

```
Normal Mode:
  gg        → Về đầu file
  G         → Xuống cuối file
  :100      → Nhảy đến dòng 100
  dd        → Xóa cả dòng hiện tại
  yy        → Copy cả dòng
  p         → Paste
  u         → Undo
  Ctrl+r    → Redo
  /keyword  → Tìm kiếm (n để tìm tiếp, N để tìm ngược)

Command Mode (gõ : trước):
  :w        → Lưu file
  :q        → Thoát (nếu chưa thay đổi)
  :wq       → Lưu và thoát
  :q!       → Thoát không lưu
  :%s/old/new/g  → Thay thế tất cả "old" bằng "new" trong file
```

### 1.13.3. Nano — Lựa chọn đơn giản hơn

```bash
nano /etc/nginx/nginx.conf

# Ctrl+O  → Lưu file
# Ctrl+X  → Thoát
# Ctrl+W  → Tìm kiếm
# Ctrl+K  → Xóa cả dòng
```

---

## 1.14. Quy Trình Thực Tế: Debug Ứng Dụng NestJS Trên Server

Dưới đây là quy trình điển hình khi ứng dụng NestJS của bạn bị lỗi trên server:

```
Nhận báo cáo lỗi
      ↓
SSH vào server
      ↓
Kiểm tra process còn chạy không
      ↓
Xem log ứng dụng
      ↓
Kiểm tra tài nguyên (RAM, Disk)
      ↓
Kiểm tra kết nối Database/Redis
      ↓
Xem log hệ thống
      ↓
Sửa lỗi và restart
```

**Thực hiện từng bước:**

```bash
# Bước 1: SSH vào server
ssh -i ~/.ssh/id_rsa ubuntu@203.0.113.10

# Bước 2: Kiểm tra process Node.js còn chạy không
ps aux | grep node
# Hoặc nếu dùng Docker:
docker ps | grep my-api

# Bước 3: Xem log ứng dụng
tail -n 200 /var/log/app/error.log
# Hoặc Docker:
docker logs my-api --tail 200

# Bước 4: Kiểm tra RAM
free -h
# Nếu hết RAM → kill process chiếm nhiều nhất hoặc restart server

# Bước 5: Kiểm tra Disk
df -h
# Nếu full disk → tìm và xóa file lớn
du -sh /var/log/* | sort -h   # Xem log file nào lớn nhất
find /var/log -name "*.log" -size +100M  # Tìm file log > 100MB

# Bước 6: Kiểm tra kết nối database
# Từ server, thử kết nối PostgreSQL:
psql -h localhost -U postgres -d myapp -c "SELECT 1"

# Bước 7: Xem log hệ thống
journalctl -n 100 --since "30 minutes ago"

# Bước 8: Restart ứng dụng
sudo systemctl restart myapp
# Hoặc Docker:
docker restart my-api

# Bước 9: Xác nhận đã hoạt động
curl http://localhost:3000/api/health
```

---

## 1.15. Script Shell Thực Tế

### 1.15.1. Script Deploy NestJS Cơ Bản

```bash
#!/bin/bash
# Tên file: deploy.sh
# Mục đích: Deploy ứng dụng NestJS lên server

# Dừng ngay nếu có lỗi
set -e

# =====================
# Cấu hình
# =====================
APP_DIR="/opt/app/my-nestjs-api"
LOG_DIR="/var/log/app"
SERVICE_NAME="my-nestjs-api"

echo "========================================="
echo "  Bắt đầu deploy: $(date)"
echo "========================================="

# Bước 1: Di chuyển vào thư mục app
cd $APP_DIR

# Bước 2: Pull code mới nhất
echo "📥 Pulling latest code..."
git pull origin main

# Bước 3: Cài đặt dependencies
echo "📦 Installing dependencies..."
npm ci --only=production

# Bước 4: Build TypeScript
echo "🔨 Building application..."
npm run build

# Bước 5: Tạo thư mục log nếu chưa có
mkdir -p $LOG_DIR

# Bước 6: Restart service
echo "🔄 Restarting service..."
sudo systemctl restart $SERVICE_NAME

# Bước 7: Chờ service khởi động
echo "⏳ Waiting for service to start..."
sleep 3

# Bước 8: Kiểm tra health
echo "🏥 Checking health..."
HEALTH=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:3000/health)
if [ "$HEALTH" = "200" ]; then
    echo "✅ Deploy thành công! Health check: OK"
else
    echo "❌ Deploy thất bại! Health check trả về: $HEALTH"
    echo "Xem log để biết thêm:"
    journalctl -u $SERVICE_NAME -n 50
    exit 1
fi

echo "========================================="
echo "  Deploy hoàn tất: $(date)"
echo "========================================="
```

```bash
# Cấp quyền chạy và chạy script
chmod +x deploy.sh
./deploy.sh
```

### 1.15.2. Script Backup Database

```bash
#!/bin/bash
# Tên file: backup-db.sh
# Mục đích: Backup PostgreSQL và xóa backup cũ hơn 7 ngày

set -e

# Cấu hình — đọc từ biến môi trường để không hard-code
DB_HOST=${DB_HOST:-"localhost"}
DB_NAME=${DB_NAME:-"myapp_prod"}
DB_USER=${DB_USER:-"postgres"}
BACKUP_DIR="/var/backups/postgres"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="$BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz"

# Tạo thư mục backup nếu chưa có
mkdir -p $BACKUP_DIR

echo "📦 Bắt đầu backup database: $DB_NAME"

# Dump và nén
pg_dump -h $DB_HOST -U $DB_USER $DB_NAME | gzip > $BACKUP_FILE

echo "✅ Backup thành công: $BACKUP_FILE ($(du -sh $BACKUP_FILE | cut -f1))"

# Xóa backup cũ hơn 7 ngày
echo "🗑️  Xóa backup cũ hơn 7 ngày..."
find $BACKUP_DIR -name "*.sql.gz" -mtime +7 -delete

echo "📊 Danh sách backup hiện tại:"
ls -lh $BACKUP_DIR
```

---

## 1.16. Best Practices

### 1.16.1. Bảo mật

```bash
# 1. Không bao giờ làm việc bằng user root trực tiếp
# Dùng sudo khi cần quyền root
sudo apt update  # ✅
# Không dùng: su - root rồi làm mọi thứ

# 2. Đặt quyền tối thiểu cần thiết
chmod 600 ~/.ssh/id_rsa          # SSH private key chỉ owner đọc được
chmod 644 .env                   # Cấu hình: owner đọc/ghi, group chỉ đọc
chmod 755 /opt/app               # Thư mục app: owner full, group/others đọc+vào

# 3. Không commit file nhạy cảm
echo ".env" >> .gitignore
echo ".env.local" >> .gitignore
echo "*.pem" >> .gitignore
echo "*.key" >> .gitignore

# 4. Dùng SSH key, không dùng password
ssh-keygen -t ed25519 -C "deploy-key"
# Tắt đăng nhập SSH bằng password trên server:
sudo vim /etc/ssh/sshd_config
# Đặt: PasswordAuthentication no
# Đặt: PermitRootLogin no
```

### 1.16.2. Hiệu quả làm việc

```bash
# 1. Dùng Tab completion
cd /var/lo<Tab>          # Tự hoàn thành thành /var/log/

# 2. Dùng lịch sử lệnh
history                  # Xem lịch sử 500 lệnh gần nhất
!200                     # Chạy lại lệnh số 200
!!                       # Chạy lại lệnh vừa gõ
Ctrl+R                   # Tìm kiếm trong lịch sử (gõ từ khóa)

# 3. Đặt alias cho lệnh hay dùng
# Thêm vào ~/.bashrc:
alias ll='ls -la'
alias gs='git status'
alias gp='git pull'
alias dc='docker-compose'
alias dps='docker ps'
alias dlogs='docker logs --tail 100 -f'

# 4. Pipe — kết hợp nhiều lệnh
ps aux | grep node | grep -v grep    # Xem node processes
cat app.log | grep ERROR | tail -50  # 50 dòng ERROR cuối
ls -la | awk '{print $5, $9}'        # Chỉ hiển thị size và tên file

# 5. Redirect output
command > file.txt       # Ghi đè file (overwrite)
command >> file.txt      # Thêm vào cuối file (append)
command 2>&1             # Gộp stderr vào stdout
command > output.txt 2>&1  # Lưu cả stdout và stderr vào file
command > /dev/null 2>&1   # Im lặng hoàn toàn (bỏ qua tất cả output)
```

### 1.16.3. Quản lý log

```bash
# 1. Luôn kiểm tra log khi có sự cố
tail -f /var/log/app/error.log

# 2. Cấu hình log rotation để tránh đầy disk
# File: /etc/logrotate.d/myapp
/var/log/app/*.log {
    daily          # Rotate mỗi ngày
    rotate 14      # Giữ 14 file cũ
    compress       # Nén file cũ
    delaycompress  # Nén từ file thứ 2 trở đi
    missingok      # Không báo lỗi nếu file không tồn tại
    notifempty     # Không rotate nếu file rỗng
}

# 3. Dùng journalctl để xem log systemd
journalctl -u myapp -f                   # Follow log
journalctl -u myapp --since "2024-01-15" # Từ ngày cụ thể
```

### 1.16.4. Kiểm tra trước khi làm

```bash
# Trước khi xóa thư mục lớn → kiểm tra kỹ
ls -la /path/to/delete    # Xem nội dung trước
du -sh /path/to/delete    # Xem kích thước

# Trước khi thay thế file quan trọng → backup
cp nginx.conf nginx.conf.bak    # Backup cấu hình nginx
# Sửa, test, nếu lỗi thì restore:
cp nginx.conf.bak nginx.conf

# Test cấu hình trước khi reload service
sudo nginx -t             # Test cấu hình nginx
sudo sshd -t              # Test cấu hình SSH
```

---

## Tóm Tắt Chương 1

| Chủ đề | Lệnh/Công cụ quan trọng | Dùng khi nào |
|---|---|---|
| Điều hướng | `pwd`, `ls`, `cd` | Hàng ngày |
| Quản lý file | `cp`, `mv`, `rm`, `mkdir` | Hàng ngày |
| Xem nội dung | `cat`, `tail -f`, `less` | Debug log |
| Tìm kiếm | `grep`, `find` | Debug, audit |
| Phân quyền | `chmod`, `chown` | Cài đặt, bảo mật |
| Process | `ps`, `kill`, `systemctl` | Debug, quản lý service |
| Network | `curl`, `ss`, `ssh`, `scp` | Deploy, test API |
| Tài nguyên | `df`, `free`, `du` | Monitor server |
| Lên lịch | `cron` | Tự động hóa |
| Soạn thảo | `vim`, `nano` | Sửa cấu hình |

---
