# Thư Mục File Cấu Hình — Demo 3 (Tích Hợp Client DNS + Web Server)

Thư mục này chứa toàn bộ các file được tạo mới và cấu hình bổ sung trong giai đoạn **Demo 3** (so với bản OVA bàn giao từ N3) để bạn dễ dàng quản lý và push lên GitHub.

---

## Cấu trúc thư mục

```text
configs-demo3/
├── README.md
├── ns1/                                       # Cấu hình trên máy Web Server (192.168.56.10)
│   ├── etc/nginx/sites-available/lab.test     # Cấu hình Virtual Host Nginx cho lab.test
│   └── var/www/lab.test/index.html            # Trang chủ web nội bộ (giao diện Card hiện đại)
└── ns2/                                       # Cấu hình trên máy Client DNS (192.168.56.11)
    └── etc/netplan/99-hostonly.yaml           # Cấu hình Netplan trỏ DNS resolver về BIND9
```

---

## Chi tiết từng file & Vị trí cài đặt

### 1. Trên máy `ns1` (Web Server)

| File trong repo | Đường dẫn trên Linux VM | Mục đích |
|---|---|---|
| `ns1/var/www/lab.test/index.html` | `/var/www/lab.test/index.html` | Trang web chào mừng khi `curl http://www.lab.test`. Thiết kế giao diện Card hiện đại với CSS. |
| `ns1/etc/nginx/sites-available/lab.test` | `/etc/nginx/sites-available/lab.test` | Nginx Server Block cấu hình lắng nghe port 80 cho các domain `www.lab.test`, `lab.test`, `web.lab.test`. |

**Lệnh kích hoạt trên `ns1`:**
```bash
sudo ln -s /etc/nginx/sites-available/lab.test /etc/nginx/sites-enabled/
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

---

### 2. Trên máy `ns2` (Client)

| File trong repo | Đường dẫn trên Linux VM | Mục đích |
|---|---|---|
| `ns2/etc/netplan/99-hostonly.yaml` | `/etc/netplan/99-hostonly.yaml` | Cấu hình IP tĩnh `192.168.56.11` cho card Host-only (`enp0s8`) và cấu hình nameservers trỏ về `192.168.56.10` (ns1) và `192.168.56.11` (ns2). |

**Lệnh áp dụng trên `ns2`:**
```bash
sudo netplan apply
```
