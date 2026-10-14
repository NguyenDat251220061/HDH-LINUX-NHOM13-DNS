# Kịch Bản Trình Bày & Hướng Dẫn Quay Video Demo 3
## Đề tài: Tích Hợp Client DNS + Web Server Nội Bộ (BIND9)

- **Người thực hiện:** N4 — Nguyễn Minh Trọng
- **Thời lượng dự kiến:** 5 – 7 phút
- **Môi trường lab:**
  - Máy trái: `ns1` (`192.168.56.10`) — Primary DNS Server + Nginx Web Server
  - Máy phải: `ns2` (`192.168.56.11`) — Secondary DNS Server đóng vai trò Client kiểm thử

---

# PHẦN 1: KỊCH BẢN CHI TIẾT (LỜI THOẠI & LỆNH GÕ)

```
Bố cục màn hình:
┌──────────────────────────────┬──────────────────────────────┐
│  TERMINAL TRÁI (ns1)         │  TERMINAL PHẢI (ns2 - Client)│
│  IP: 192.168.56.10           │  IP: 192.168.56.11           │
└──────────────────────────────┴──────────────────────────────┘
```

---

### Chặng 1: Giới thiệu tổng quan & Kiểm tra kết nối (1 phút)

🗣️ **Lời thoại:**
> *"Kính chào thầy cô và các bạn, em là Nguyễn Minh Trọng, đại diện thành viên N4. Hôm nay em xin demo phần 3: Tích hợp Client DNS và Web Server nội bộ.  
> Ở hai phần trước, nhóm đã hoàn thành cấu hình Primary DNS trên máy ns1 và Secondary DNS trên máy ns2. Trong phần này, em sẽ đóng vai trò Client trên máy ns2, chứng minh quá trình phân giải tên miền truy cập vào Web Server được đặt tại máy ns1, giải thích cơ chế ưu tiên giữa file hosts và DNS, và chứng minh tính năng DNS Caching."*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
# 1. Kiểm tra IP Client
ip -br a

# 2. Ping kiểm tra thông mạng với Web Server ns1
ping -c 3 192.168.56.10
```

🗣️ **Lời thoại:**
> *"Trước tiên, máy Client ns2 có IP là 192.168.56.11 trên card enp0s8 và ping tới Web Server 192.168.56.10 hoàn toàn thông suốt với 0% packet loss."*

---

### Chặng 2: Client phân giải DNS và truy cập Web (2 phút)

🗣️ **Lời thoại:**
> *"Tiếp theo, em kiểm tra cấu hình DNS resolver của Client bằng `resolvectl` xem đã trỏ về cụm BIND9 chưa."*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
# Xem DNS Server mà Client đang sử dụng
resolvectl status enp0s8 | grep "DNS Servers"
```
👉 *Kết quả hiện:* `DNS Servers: 192.168.56.10 192.168.56.11`

🗣️ **Lời thoại:**
> *"Client đang trỏ về đúng 2 máy chủ DNS nội bộ. Bây giờ em thực hiện các truy vấn DNS xuôi, ngược và CNAME."*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
# Truy vấn A Record
dig www.lab.test +short

# Truy vấn CNAME Record
dig web.lab.test +short

# Truy vấn Reverse DNS (PTR)
dig -x 192.168.56.10 +short
```

🗣️ **Lời thoại:**
> *"Như thầy cô thấy, `www.lab.test` trỏ đúng về IP 192.168.56.10, bí danh `web.lab.test` chuyển hướng đúng về www, và phân giải ngược ra `ns1.lab.test`.  
> Bây giờ Client sẽ sử dụng lệnh `curl` để gửi HTTP request truy cập web thông qua tên miền vừa phân giải."*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
curl http://www.lab.test
curl http://web.lab.test
```
👉 *Màn hình in ra nội dung mã HTML của trang Web `lab.test`.*

🗣️ **Lời thoại:**
> *"Cả hai domain đều phản hồi trang web thành công. Điều này chứng minh Client đã tích hợp trơn tru với hệ thống BIND9 DNS và Web Server Nginx."*

---

### Chặng 3: Chứng minh `/etc/hosts` ưu tiên hơn DNS (2 phút)

🗣️ **Lời thoại:**
> *"Bây giờ em sẽ thực hiện một trường hợp thực tế quan trọng: Cơ chế ưu tiên phân giải của hệ điều hành Linux được định nghĩa trong file `/etc/nsswitch.conf`."*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
# Kiểm tra thứ tự phân giải tên miền của OS
grep hosts /etc/nsswitch.conf
```
👉 *Kết quả hiện:* `hosts: files dns`

🗣️ **Lời thoại:**
> *"Thứ tự là `files` rồi mới đến `dns`. Tức là Linux sẽ tra cứu file `/etc/hosts` trước khi hỏi máy chủ DNS. Để chứng minh, em sẽ giả mạo một bản ghi trỏ tên miền `www.lab.test` về một địa chỉ IP ảo không tồn tại là `192.168.56.99`."*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
# Thêm bản ghi giả mạo
echo "192.168.56.99 www.lab.test" | sudo tee -a /etc/hosts

# Kiểm tra lại file hosts
cat /etc/hosts | grep lab.test
```

🗣️ **Lời thoại:**
> *"Bây giờ em thử ping và truy cập web bằng curl:"*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
# Ping kiểm tra IP
ping -c 2 www.lab.test

# Thử truy cập web
curl --connect-timeout 3 http://www.lab.test
```
👉 *Ping trỏ về `192.168.56.99` và bị timeout. Curl báo lỗi `Connection timed out`.*

🗣️ **Lời thoại:**
> *"Mọi yêu cầu duyệt web và ping đều bị thất bại do hệ điều hành đã ưu tiên lấy IP từ file hosts.  
> TUY NHIÊN, nếu người quản trị dùng lệnh `dig` hỏi trực tiếp tới máy chủ BIND9 trên ns1:"*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
dig @192.168.56.10 www.lab.test +short
```
👉 *Kết quả trả về vẫn là: `192.168.56.10`.*

🗣️ **Lời thoại:**
> *"Lệnh `dig` giao tiếp trực tiếp với DNS Server qua port 53 nên hoàn toàn không bị ảnh hưởng bởi file `/etc/hosts`.  
> Bây giờ em sẽ xóa dòng giả mạo này đi để đưa hệ sinh thái mạng về trạng thái bình thường."*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
# Xóa bản ghi giả mạo
sudo sed -i '/192.168.56.99/d' /etc/hosts

# Kiểm tra lại curl
curl http://www.lab.test
```
👉 *Website hoạt động trở lại bình thường.*

---

### Chặng 4: Demo DNS Caching trên Client (1.5 phút)

🗣️ **Lời thoại:**
> *"Phần cuối cùng, em xin trình bày về cơ chế DNS Caching trên Client thông qua dịch vụ `systemd-resolved`."*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
# 1. Xóa sạch bộ nhớ đệm
sudo resolvectl flush-caches

# 2. Xem thống kê ban đầu
resolvectl statistics
```
👉 *Chỉ vào dòng `Current Cache Size: 0`.*

🗣️ **Lời thoại:**
> *"Hiện tại bộ nhớ đệm đang bằng 0. Bây giờ em thực hiện truy vấn lần thứ nhất:"*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
dig www.lab.test
resolvectl statistics
```
👉 *Chỉ vào: `Current Cache Size: 1` (hoặc số > 0) và `Cache Misses: tăng thêm 1`.*

🗣️ **Lời thoại:**
> *"Lần truy vấn đầu tiên được ghi nhận là Cache Miss vì Client chưa có dữ liệu và phải gửi yêu cầu sang DNS Server ns1. Đồng thời bản ghi đã được nạp vào RAM.  
> Bây giờ em thực hiện truy vấn lần thứ hai:"*

⌨️ **Thao tác trên Terminal PHẢI (`ns2`):**
```bash
dig www.lab.test
resolvectl statistics
```
👉 *Chỉ vào dòng: `Cache Hits: tăng thêm 1`.*

🗣️ **Lời thoại:**
> *"Chỉ số `Cache Hits` đã tăng lên, chứng minh câu trả lời lần này được trả về tức thì từ bộ nhớ RAM cục bộ của máy Client mà không cần tốn băng thông truy vấn lại DNS Server."*

---

### Chặng 5: Tổng kết (30 giây)

🗣️ **Lời thoại:**
> *"Như vậy em đã hoàn thành toàn bộ kịch bản của Demo 3:  
> 1. Kết nối thành công Client tới Web Server thông qua hệ thống BIND9 DNS.  
> 2. Chứng minh thực nghiệm thứ tự ưu tiên của `/etc/hosts` so với DNS qua `nsswitch.conf`.  
> 3. Chứng minh hoạt động của DNS Caching trên hệ điều hành Linux.  
> Em xin chân thành cảm ơn thầy cô và các bạn đã lắng nghe!"*

---

# PHẦN 2: HƯỚNG DẪN QUAY VIDEO DEMO (CHẤT LƯỢNG CAO)

### 1. Chuẩn bị phần mềm quay màn hình
- **Khuyên dùng:** **OBS Studio** (Miễn phí, không watermark, chất lượng nét nhất) hoặc **Xbox Game Bar** (phím tắt `Windows + G` có sẵn trên Windows 10/11).
- **Cài đặt video trong OBS:**
  - Base Resolution: `1920x1080` (Full HD).
  - Framerate: `60 FPS` (hoặc 30 FPS).
  - Audio: Sử dụng tai nghe có mic để khử tiếng ồn môi trường, âm thanh rõ ràng.

### 2. Thiết lập giao diện màn hình quay
- **Cách sắp xếp cửa sổ:**
  - Mở 2 cửa sổ VirtualBox hoặc 2 cửa sổ SSH Client (như MobaXterm / Windows Terminal / PuTTY).
  - Bấm phím `Windows + Phím mũi tên Trái` cho cửa sổ `ns1`.
  - Bấm phím `Windows + Phím mũi tên Phải` cho cửa sổ `ns2`.
- **Cỡ chữ (Font size):**
  - Chỉnh font chữ Terminal lên kích thước **14pt hoặc 16pt** để người xem trên điện thoại hoặc máy chiếu có thể đọc rõ từng dòng lệnh.
  - Xóa sạch màn hình trước khi bắt đầu bằng lệnh `clear`.

### 3. Checklist trước khi bấm "Record"
- [ ] Cả 2 máy `ns1` và `ns2` đã khởi động và đăng nhập sẵn sàng.
- [ ] Đã chạy test thử một vòng: `ping`, `curl`, `dig` đều chạy ngon lành.
- [ ] Đảm bảo file `/etc/hosts` trên `ns2` đang **sạch** (không còn sót IP `.99`).
- [ ] Chạy lệnh `sudo resolvectl flush-caches` để reset cache về 0.
- [ ] Chuẩn bị sẵn file kịch bản này ở màn hình phụ hoặc điện thoại để nói theo mạch lạc.

### 4. Mẹo khi quay video
- **Nếu lỡ gõ sai lệnh hoặc nói vấp:**
  - **ĐỪNG DỪNG VIDEO!** Hãy im lặng khoảng 2 giây, gõ `clear` màn hình rồi đọc lại câu lệnh đó.
  - Khi quay xong, dùng phần mềm cắt video đơn giản (như CapCut PC hoặc Clipchamp có sẵn trên Windows) cắt bỏ đoạn 2 giây lỗi đó đi là video sẽ hoàn hảo.
- Giữ tốc độ gõ phím và nói vừa phải, dứt khoát, để người xem kịp quan sát kết quả terminal in ra.
