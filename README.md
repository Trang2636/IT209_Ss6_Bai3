# Bài 3: Cấu hình tường lửa UFW và chuẩn đoán cổng mạng

## 1. Mục tiêu

- Sử dụng công cụ tường lửa **UFW** (Uncomplicated Firewall) để bảo vệ máy chủ.
- Thiết lập quy tắc chặn/mở cổng mạng phục vụ deploy ứng dụng an toàn.
- Sử dụng các công cụ chẩn đoán mạng (`ss`, `curl`, `ufw status`) để kiểm tra cổng kết nối.

---

## 2. Các bước thực hiện

### Bước 1: Cài đặt UFW (nếu chưa có)

```bash
sudo apt update && sudo apt install ufw -y
```

Kiểm tra phiên bản:

```bash
sudo ufw version
```

---

### Bước 2: Cấu hình chính sách mặc định

Chặn **toàn bộ** kết nối đi vào (deny incoming):

```bash
sudo ufw default deny incoming
```

**Kết quả:**

```
Default incoming policy changed to 'deny'
(be sure to update your rules accordingly)
```

Cho phép **toàn bộ** kết nối đi ra (allow outgoing):

```bash
sudo ufw default allow outgoing
```

**Kết quả:**

```
Default outgoing policy changed to 'allow'
(be sure to update your rules accordingly)
```

> **Giải thích chính sách mặc định:**
>
> | Chính sách | Ý nghĩa |
> |---|---|
> | `deny incoming` | Mặc định chặn mọi kết nối từ bên ngoài vào — bảo vệ tối đa |
> | `allow outgoing` | Cho phép máy chủ kết nối ra ngoài (cập nhật, tải gói, gọi API,...) |

---

### Bước 3: Mở cổng SSH (port 22)

```bash
sudo ufw allow 22/tcp
```

**Kết quả:**

```
Rules updated
Rules updated (v6)
```

> ⚠️ **QUAN TRỌNG:** Bước này **bắt buộc** phải thực hiện **TRƯỚC** khi bật UFW. Nếu không mở port 22, bạn sẽ bị mất kết nối SSH và không thể quản trị máy chủ từ xa.

---

### Bước 4: Mở cổng ứng dụng Web (port 8080)

```bash
sudo ufw allow 8080/tcp
```

**Kết quả:**

```
Rules updated
Rules updated (v6)
```

---

### Bước 5: Kích hoạt tường lửa UFW

```bash
sudo ufw enable
```

**Kết quả:**

```
Command may disrupt existing ssh connections. Proceed with operation (y|n)? y
Firewall is active and enabled on system startup
```

> Nhấn `y` để xác nhận. UFW sẽ tự động khởi động cùng hệ thống sau mỗi lần boot.

---

## 3. Kiểm tra kết quả

### 3.1. Xem trạng thái tường lửa chi tiết

```bash
sudo ufw status verbose
```

**Kết quả mong đợi:**

```
Status: active
Logging: on (low)
Default: deny (incoming), allow (outgoing), disabled (routed)
New profiles: skip

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW IN    Anywhere
8080/tcp                   ALLOW IN    Anywhere
22/tcp (v6)                ALLOW IN    Anywhere (v6)
8080/tcp (v6)              ALLOW IN    Anywhere (v6)
```

> **Phân tích kết quả:**
>
> | Thông tin | Giá trị | Ý nghĩa |
> |---|---|---|
> | Status | `active` | Tường lửa đang hoạt động |
> | Default incoming | `deny` | Mặc định chặn mọi kết nối đến |
> | Default outgoing | `allow` | Cho phép mọi kết nối đi ra |
> | Port 22/tcp | `ALLOW IN` | Cho phép SSH từ mọi nguồn |
> | Port 8080/tcp | `ALLOW IN` | Cho phép truy cập ứng dụng web từ mọi nguồn |
> | (v6) | IPv6 rules | Quy tắc tương tự cho IPv6 |

---

### 3.2. Kiểm tra các cổng đang lắng nghe trên máy chủ

```bash
ss -tlnp
```

**Kết quả mong đợi:**

```
State    Recv-Q   Send-Q   Local Address:Port   Peer Address:Port   Process
LISTEN   0        128      0.0.0.0:22            0.0.0.0:*           users:(("sshd",pid=1234,fd=3))
LISTEN   0        128      0.0.0.0:8080          0.0.0.0:*           users:(("java",pid=5678,fd=5))
LISTEN   0        128      [::]:22               [::]:*              users:(("sshd",pid=1234,fd=4))
LISTEN   0        128      [::]:8080             [::]:*              users:(("java",pid=5678,fd=6))
```

> **Giải thích tham số `ss -tlnp`:**
>
> | Tham số | Ý nghĩa |
> |---|---|
> | `-t` | Chỉ hiển thị kết nối TCP |
> | `-l` | Chỉ hiển thị các socket đang ở trạng thái LISTEN |
> | `-n` | Hiển thị số port thay vì tên dịch vụ |
> | `-p` | Hiển thị thông tin tiến trình (process) đang sở hữu socket |

---

### 3.3. Kiểm tra kết nối từ bên ngoài bằng `curl`

Từ một máy khác hoặc local, kiểm tra kết nối đến port 8080:

```bash
curl -I http://<IP_SERVER>:8080
```

**Kết quả mong đợi (nếu ứng dụng đang chạy):**

```
HTTP/1.1 200 OK
Content-Type: text/html
...
```

Kiểm tra port bị chặn (ví dụ port 3306 - MySQL):

```bash
curl -I http://<IP_SERVER>:3306 --connect-timeout 5
```

**Kết quả mong đợi:** Kết nối bị timeout vì port 3306 không nằm trong danh sách allow.

---

### 3.4. Xem danh sách quy tắc đánh số (hữu ích khi cần xoá rule)

```bash
sudo ufw status numbered
```

**Kết quả:**

```
Status: active

     To                         Action      From
     --                         ------      ----
[ 1] 22/tcp                     ALLOW IN    Anywhere
[ 2] 8080/tcp                   ALLOW IN    Anywhere
[ 3] 22/tcp (v6)                ALLOW IN    Anywhere (v6)
[ 4] 8080/tcp (v6)              ALLOW IN    Anywhere (v6)
```

> Để xoá một rule, sử dụng: `sudo ufw delete <số thứ tự>`, ví dụ `sudo ufw delete 2`.

---

## 4. Các lệnh UFW hữu ích bổ sung

| Lệnh | Chức năng |
|---|---|
| `sudo ufw status` | Xem trạng thái ngắn gọn |
| `sudo ufw status verbose` | Xem trạng thái chi tiết |
| `sudo ufw status numbered` | Xem rules đánh số |
| `sudo ufw allow <port>/tcp` | Mở port TCP |
| `sudo ufw deny <port>/tcp` | Chặn port TCP |
| `sudo ufw delete <rule_number>` | Xoá rule theo số thứ tự |
| `sudo ufw allow from <IP> to any port <port>` | Mở port cho IP cụ thể |
| `sudo ufw disable` | Tắt tường lửa |
| `sudo ufw reset` | Reset toàn bộ cấu hình về mặc định |
| `sudo ufw reload` | Nạp lại cấu hình |

---

## 5. Sơ đồ luồng kết nối qua UFW

```
Internet/Client
       │
       ▼
┌──────────────────────────────┐
│        UFW Firewall          │
│                              │
│  Default: DENY incoming      │
│  Default: ALLOW outgoing     │
│                              │
│  Rules:                      │
│  ✅ Port 22/tcp  → ALLOW IN  │
│  ✅ Port 8080/tcp → ALLOW IN │
│  ❌ Mọi port khác → DENY    │
│                              │
└──────────┬───────────────────┘
           │
           ▼
┌──────────────────────────────┐
│       Linux Server           │
│                              │
│  SSH Server  ← Port 22       │
│  Web App     ← Port 8080     │
│                              │
└──────────────────────────────┘
```

---

## 6. Tổng kết

| Hạng mục | Kết quả |
|---|---|
| Chính sách mặc định: deny incoming | ✅ Đã cấu hình |
| Chính sách mặc định: allow outgoing | ✅ Đã cấu hình |
| Mở port SSH (22/tcp) | ✅ ALLOW IN Anywhere |
| Mở port Web App (8080/tcp) | ✅ ALLOW IN Anywhere |
| Kích hoạt UFW | ✅ Status: active |
| Kiểm tra bằng `ss -tlnp` | ✅ Port 22 & 8080 đang LISTEN |
| Các port khác bị chặn | ✅ Đúng kỳ vọng |
