# Demo 3 — Tích hợp Client DNS + Web Server nội bộ

> Người phụ trách: **N4 — Nguyễn Minh Trọng**  
> Yêu cầu: OVA từ N3 (`ns1` và `ns2` đã hoàn tất Zone Transfer)  
> Môi trường mạng:
> - **NAT (enp0s3):** Ra Internet tải package
> - **Host-only (enp0s8):** Mạng nội bộ bài lab
>   - `ns1`: `192.168.56.10` (Primary DNS + Nginx Web Server)
>   - `ns2`: `192.168.56.11` (Secondary DNS + Client kiểm thử)

---

## Bước 0: Import OVA và kiểm tra hạ tầng

### 0.1 Import VM trong VirtualBox
1. Mở VirtualBox → **File** → **Import Appliance** → chọn file `.ova` của `ns1` và `ns2`.
2. Tại mục **MAC Address Policy**: chọn **"Generate new MAC addresses for all network adapters"** (hoặc để mặc định).
3. Kiểm tra card mạng của cả 2 VM:
   - **Adapter 1:** NAT (card `enp0s3`)
   - **Adapter 2:** Host-only (card `enp0s8`, chọn Host-Only Ethernet Adapter của máy bạn).
4. Khởi động 2 máy, đăng nhập user `dns`.

### 0.2 Verify kết nối mạng nội bộ
```bash
# Trên ns1:
ip -br a                                  # enp0s8 phải hiển thị 192.168.56.10
ping -c 4 192.168.56.11                   # ping kiểm tra kết nối tới ns2

# Trên ns2:
ip -br a                                  # enp0s8 phải hiển thị 192.168.56.11
ping -c 4 192.168.56.10                   # ping kiểm tra kết nối tới ns1
dig @192.168.56.11 www.lab.test +short     # kiểm tra Zone Transfer (phải ra 192.168.56.10)
```

> **Ghi chú:** Trên Linux lệnh `ping` chạy liên tục, dùng thêm cờ `-c 4` để gửi đúng 4 gói tin rồi tự ngắt (tương tự như Windows).

### 0.3 Chụp Snapshot dự phòng
Trước khi can thiệp cấu hình, chụp snapshot trên VirtualBox để có thể rollback bất kỳ lúc nào:
- `ns1` → Snapshot: `truoc-demo3`
- `ns2` → Snapshot: `truoc-demo3`

---

## Bước 1: Cài đặt Nginx trên ns1 (Web Server nội bộ)

Thực hiện trên máy **`ns1`**:

```bash
# Cập nhật repo và cài đặt Nginx
sudo apt update
sudo apt install -y nginx
```

### 1.1 Tạo thư mục web và file `index.html`

```bash
sudo mkdir -p /var/www/lab.test
sudo nano /var/www/lab.test/index.html
```

Dán nội dung sau vào file:
```html
<!DOCTYPE html>
<html lang="vi">
<head>
    <meta charset="UTF-8">
    <title>lab.test — DNS Demo</title>
    <style>
        body {
            font-family: 'Segoe UI', sans-serif;
            background: #1a1a2e;
            color: #eee;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .card {
            background: #16213e;
            border: 1px solid #0f3460;
            border-radius: 12px;
            padding: 40px 60px;
            text-align: center;
            box-shadow: 0 0 30px rgba(15, 52, 96, 0.5);
        }
        h1 { color: #e94560; margin-bottom: 10px; }
        p  { color: #a3b1c6; }
        code { background: #0f3460; padding: 2px 8px; border-radius: 4px; }
    </style>
</head>
<body>
    <div class="card">
        <h1>🌐 lab.test</h1>
        <p>Web server nội bộ — phân giải qua <strong>BIND9 DNS</strong></p>
        <p>Hostname: <code>ns1</code> &bull; IP: <code>192.168.56.10</code></p>
        <p>Demo 3 — Tích hợp Client</p>
    </div>
</body>
</html>
```
*(Nhấn `Ctrl + O` → `Enter` để lưu, `Ctrl + X` để thoát nano).*

### 1.2 Tạo Nginx Server Block cho domain `lab.test`

```bash
sudo nano /etc/nginx/sites-available/lab.test
```

Nội dung cấu hình:
```nginx
server {
    listen 80;
    server_name www.lab.test lab.test web.lab.test;

    root /var/www/lab.test;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### 1.3 Kích hoạt website và reload Nginx

```bash
# Tạo symbolic link kích hoạt site
sudo ln -s /etc/nginx/sites-available/lab.test /etc/nginx/sites-enabled/

# Tắt site default để tránh xung đột
sudo rm -f /etc/nginx/sites-enabled/default

# Kiểm tra cú pháp cấu hình Nginx
sudo nginx -t

# Nạp lại cấu hình và bật tự khởi động cùng OS
sudo systemctl reload nginx
sudo systemctl enable nginx
```

### 1.4 Kiểm tra Nginx cục bộ ngay trên `ns1`
```bash
curl http://localhost
```
*Kết quả in ra nội dung mã HTML của trang lab.test là hoàn tất cài đặt Web Server.*

---

## Bước 2: Cấu hình ns2 làm Client DNS (Sử dụng Netplan + systemd-resolved)

Thực hiện trên máy **`ns2`**:

### 2.1 Cấu hình Netplan cho card `enp0s8`

Kiểm tra và sửa file Netplan cho card Host-only:
```bash
sudo nano /etc/netplan/99-hostonly.yaml
```

Nhập cấu hình chính xác cho card `enp0s8`:
```yaml
network:
  version: 2
  ethernets:
    enp0s8:
      addresses:
        - 192.168.56.11/24
      nameservers:
        addresses:
          - 192.168.56.10      # ns1 — Primary DNS
          - 192.168.56.11      # ns2 — Secondary DNS
        search:
          - lab.test            # Tự động gắn đuôi .lab.test khi gõ tên ngắn
      optional: true
```

Áp dụng cấu hình mạng:
```bash
sudo netplan apply
```

*(Lưu ý: Nếu Netplan báo lỗi `Cannot find unique matching interface for enp0s3`, kiểm tra file `00-installer-config.yaml` và xóa khối `match: macaddress:...` như đã xử lý).*

### 2.2 Kiểm tra kết nối phân giải DNS từ Client
Kiểm tra xem `systemd-resolved` đã nhận đúng DNS Server chưa:
```bash
resolvectl status enp0s8
# Mục "DNS Servers" phải hiện: 192.168.56.10 và 192.168.56.11
```

Thực hiện truy vấn DNS:
```bash
# Phân giải tên miền xuôi
dig www.lab.test +short
# Kết quả ra: 192.168.56.10

# Phân giải ngược từ IP ra domain
dig -x 192.168.56.10 +short
# Kết quả ra: ns1.lab.test.

# Kiểm tra CNAME
dig web.lab.test +short
# Kết quả ra: www.lab.test. -> 192.168.56.10
```

### 2.3 Truy cập Web Server từ Client bằng tên miền
```bash
curl http://www.lab.test
# Phải hiển thị nội dung HTML của trang lab.test

curl http://web.lab.test
# Cũng phải hiển thị (vì web là CNAME -> www)
```

---

## Bước 3: Minh họa `/etc/hosts` ưu tiên hơn DNS

Đây là phần trọng tâm chứng minh cơ chế phân giải tên miền của Linux theo `/etc/nsswitch.conf`.

### 3.1 Kiểm tra cơ chế ưu tiên hệ thống
```bash
grep hosts /etc/nsswitch.conf
```
*Kết quả:* `hosts: files dns`  
→ `files` (`/etc/hosts`) được tra cứu **trước** `dns`.

### 3.2 Tạo bản ghi giả mạo trong `/etc/hosts` (trên ns2)
```bash
sudo nano /etc/hosts
```
Thêm dòng sau vào cuối file:
```text
192.168.56.99   www.lab.test
```
*(IP `192.168.56.99` là IP không tồn tại).*

### 3.3 Chứng minh `/etc/hosts` ghi đè kết quả DNS
```bash
# 1. ping dùng thư viện nsswitch -> bị /etc/hosts điều hướng sang IP sai .99
ping -c 2 www.lab.test
# Kết quả: PING www.lab.test (192.168.56.99) -> Timeout / Destination Host Unreachable

# 2. curl cũng dùng thư viện hệ thống -> kết nối thất bại
curl --connect-timeout 3 http://www.lab.test
# Kết quả: Connection timed out / Failed to connect

# 3. dig hỏi trực tiếp DNS Server BIND9 (bỏ qua /etc/hosts hoàn toàn)
dig @192.168.56.10 www.lab.test +short
# Kết quả: 192.168.56.10 (Vẫn trả về IP thật của Web Server!)
```

> 💡 **Điểm nhấn lý thuyết:**  
> - Các ứng dụng như `curl`, `ping`, trình duyệt duyệt web tuân theo thứ tự trong `/etc/nsswitch.conf` nên bị file `/etc/hosts` can thiệp.  
> - Tiện ích quản trị mạng `dig` khi chỉ định `@<DNS_IP>` sẽ gửi gói tin trực tiếp tới DNS Server trên cổng 53, hoàn toàn bỏ qua `/etc/hosts`.

### 3.4 Khôi phục trạng thái chuẩn
```bash
sudo nano /etc/hosts
# Xóa bỏ dòng "192.168.56.99   www.lab.test" vừa thêm

# Kiểm tra lại
ping -c 2 www.lab.test      # Trỏ về 192.168.56.10 -> OK
curl http://www.lab.test     # Nhận lại trang web bình thường -> OK
```

---

## Bước 4: Demo DNS Caching với `systemd-resolved`

Chứng minh cơ chế lưu bộ nhớ đệm (Cache) trên Client giúp tối ưu tốc độ phân giải.

### 4.1 Xóa sạch cache trước khi demo
```bash
sudo resolvectl flush-caches
resolvectl statistics
# Quan sát mục "Current Cache Size" đang ở mức: 0
```

### 4.2 Truy vấn lần 1 (Cache Miss)
```bash
dig www.lab.test
resolvectl statistics
# Quan sát:
# - "Current Cache Size" tăng lên (đã nạp bản ghi vào RAM)
# - "Cache Misses" tăng thêm 1
```

### 4.3 Truy vấn lần 2 (Cache Hit)
```bash
dig www.lab.test
resolvectl statistics
# Quan sát:
# - "Cache Hits" tăng lên thêm 1 (truy vấn được giải quyết ngay trong cache cục bộ)
```

### 4.4 Xóa lại cache để đưa hệ thống về ban đầu
```bash
sudo resolvectl flush-caches
resolvectl statistics
```

---

## Bước 5: Hoàn tất và Export OVA

1. Dọn dẹp log tạm trên cả 2 máy:
```bash
sudo apt clean
sudo journalctl --vacuum-time=1d
```
2. Đảm bảo file `/etc/hosts` trên `ns2` không còn chứa bản ghi giả `.99`.
3. Tắt máy:
```bash
sudo poweroff
```
4. Chụp Snapshot VirtualBox: `san-sang-demo`.
5. Export Appliance (OVA 2.0) với tùy chọn MAC Address Policy: **Strip all network adapter MAC addresses** hoặc **Generate new MAC addresses**.
