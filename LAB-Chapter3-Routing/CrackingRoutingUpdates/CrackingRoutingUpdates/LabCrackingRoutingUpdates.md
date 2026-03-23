# 🔓 Lab: Cracking Routing Updates (Routing Authentication Hash)

## 🎯 Mục tiêu bài Lab
Nhiều giao thức định tuyến (như OSPF, EIGRP, RIPv2) hỗ trợ cơ chế xác thực (Authentication) để ngăn chặn các Router giả mạo tham gia vào mạng. Tuy nhiên, nếu cấu hình sử dụng thuật toán băm yếu (như MD5) kết hợp với mật khẩu đơn giản, kẻ tấn công có thể bắt gói tin cập nhật định tuyến và tiến hành bẻ khóa (crack) offline để lấy mật khẩu.

## ⚙️ Môi trường & Công cụ
* **Nền tảng:** GNS3
* **Thiết bị:** Cisco Routers (đã cấu hình định tuyến và bật xác thực MD5)
* **Máy tấn công:** Kali Linux
* **Công cụ:** Wireshark, `ettercap`, `john` (John the Ripper) hoặc `hashcat`, từ điển (Wordlist - vd: `rockyou.txt`)

## 🚀 Kịch bản tấn công (Attack Scenario)

### Bước 1: Trinh sát và Bắt gói tin (Sniffing)
Kẻ tấn công kết nối vào mạng và lắng nghe các gói tin Multicast của các giao thức định tuyến (Ví dụ OSPF: `224.0.0.5`, EIGRP: `224.0.0.10`).
Sử dụng Wireshark hoặc `tcpdump` lưu lại file `.pcap`.

### Bước 2: Trích xuất Hash định tuyến
Sử dụng công cụ để trích xuất hàm băm MD5 từ gói tin pcap thu được để chuẩn bị cho quá trình crack. Có thể dùng các script Python hỗ trợ hoặc Ettercap.

### Bước 3: Crack mật khẩu (Brute-force / Dictionary Attack)
Sử dụng John the Ripper hoặc Hashcat kết hợp với Wordlist để bẻ khóa offline.

**Thực thi bằng Hashcat (Ví dụ với OSPF MD5):**
```bash
# Trích xuất hash từ pcap (giả sử dùng một script/công cụ chuyển đổi sang định dạng hashcat)
# Chạy hashcat với mode tương ứng (vd: 11400 cho SIP/OSPF, tuỳ thuộc vào module trích xuất)
hashcat -m [Mã_Mode] -a 0 routing_hash.txt /usr/share/wordlists/rockyou.txt
```

## 🛡️ Biện pháp phòng thủ (Mitigation)
Để bảo vệ các bản cập nhật định tuyến khỏi bị bẻ khóa:

1. **Sử dụng mật khẩu mạnh:** Tránh các mật khẩu có trong từ điển.
2. **Nâng cấp thuật toán băm:** Không sử dụng định dạng Cleartext hay MD5. Trên các thiết bị Cisco IOS mới, hãy cấu hình xác thực bằng HMAC-SHA-256.
   ```bash
   Router(config)# key chain OSPF_KEYS
   Router(config-keychain)# key 1
   Router(config-keychain-key)# cryptographic-algorithm hmac-sha-256
   Router(config-keychain-key)# key-string [Mật_Khẩu_Mạnh_Và_Phức_Tạp]
   ```
3. **Cấu hình Passive Interface:** Tắt việc gửi các gói tin định tuyến trên các cổng hướng ra người dùng cuối (Access/LAN), chỉ chạy định tuyến trên các cổng Point-to-Point nối giữa các Router.
   ```bash
   Router(config)# router ospf 1
   Router(config-router)# passive-interface default
   Router(config-router)# no passive-interface [Cổng_nối_với_Router_khác]
   ```
---
*Project by Nguyen Gia Bao (Hoa Bảo Lục) - 2026*