> Dựa trên 6 domain chính của khóa BTL1: Security Fundamentals, Phishing Analysis, Threat Intelligence, Digital Forensics, SIEM Analysis, Incident Response
# Kịch bản 1: Phishing Email -> Initial Access
**Mục tiêu**: Mô phỏng tấn công phishing để người dùng click link độc hại.
**Các bước thực hiện:**
## Bước 1: Chuẩn bị phishing page (trên VM-ATTACKER)
``` bash
# Clone một trang đăng nhập giả (ví dụ: Office 365, Google)
git clone https://github.com/securestep/office365_phishing.git
cd office365_phishing
# Chỉnh sửa file index.html để thay đổi nội dung
```

## Bước 2: Thiết lập GoPhish server
``` bash
cd ~/gophish
./gophish
# Truy cập https://localhost:3333
# Default: admin/gophish
```

**Cấu hình trong GoPhish:**

- **Sending Profile**: Cấu hình SMTP (dùng dịch vụ gửi mail thử nghiệm)
- **Email Template**: Tạo email phishing với nội dung giả mạo
- **Landing Page**: Trỏ đến phishing page đã tạo
- **Campaign**: Gửi đến danh sách nạn nhân (VM-WIN-ENDPOINT, VM-LINUX-ENDPOINT)

## Bước 3: Victim click link
- Victim nhận email -> click link -> nhập thông tin đăng nhập
- GoPhish ghi nhận credentials

## Bước 4: SOC phát hiện
- Splunk nhận log từ endpoint: Event ID 4688 (Process Creation) khi mở trình duyệt
- Network log từ pfSense: Kết nối đến domain lạ
- Queries Splunk:
``` sql
index=windows EventCode=4688 Process_Name="chrome.exe" OR Process_Name="firefox.exe"
| table _time, user, Process_Name, CommandLine

index=network src_ip=192.168.56.20 dest_ip=!192.168.56.*
| stats count by dest_ip, dest_port
```