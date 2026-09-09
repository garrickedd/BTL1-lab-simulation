# Quy trình xử lý phía SOC
> Dựa trên quy trình 6 pha của Incident Response trong BTL1 và các quy trình chuẩn của SOC L1

## 1. Quy trình SOC L1 - Triage & Xử lý Alert
### Phase 1: Preparation (Chuẩn bị)
- Checklist hàng ngày:
    - ✅ Kiểm tra Splunk Dashboard có hoạt động không
    - ✅ Kiểm tra các data source đang forward log không
    - ✅ Xem xét các alert đã được tạo từ hôm trước
    - ✅ Cập nhật Threat Intelligence feeds [BỔ SUNG SAU]

### Phase 2: Detection & Analysis (Phát hiện & Phân tích)
**Khi nhận được alert, SOC L1 phát hiện:**
**Bước 1 - Xác minh Alert:**
- Alert có phải false positive không?
- Context của alert là gì? (thời gian, user, IP, asset)

**Bước 2 - Thu thập thông tin:**
```sql
# Ví dụ: Điều tra alert về nhiều failed logon
index=windows EventCode=4625
| stats count by src_ip, user, dest_ip
| sort - count
```

**Bước 3 - Enrichment (làm giàu thông tin):**
- Tra cứu IP đáng ngờ trên VirusTotal, AbuseIPDB
- Tra cứu domain trên WHOIS, Passive DNS
- Kiểm tra user có phải là user hợp lệ không
- Kiểm tra asset có đang bị patch không

** Bước 4 - Phân tích sâu hơn nếu cần:**
- Xem PCAP để phân tích network (nếu có)
- Kiểm tra memory/disk forensics nếu nghi ngờ compromise
- Phân tích email nếu là phishing

### Phase 3: Containment (Ngăn chặn)
**Các hành động của SOC L1 (sau khi xác nhận incident):**
| Hành động    | Cách thực hiện    | Ghi chú    |
| ------- | ------- | ------- |
| Cô lập endpoint    | Block IP/MAC trên firewall    | pfSense rule block    |
| Khóa tài khoản    | Disable user account trong AD    | Nếu có AD    |
| Reset password    | Force password change    | Cho user bị ảnh hưởng    |
| Chặn C2 domain    | Block trên DNS/proxy    | Thêm vào blacklist    |
| Chặn IP attacker    | Firewall block rule    | Trên pfSense    |

**pfSense block rule:**
```
Action: Block
Interface: LAN
Source: 192.168.56.100
Destination: Any
```

### Phase 4: Eradication (Loại bỏ)
**Các bước loại bỏ threat:**
**1. Windows Endpoint:**
- Kill process độc hại: `taskkill /PID <pid> /F`
- Xóa file malware: `del /f C:\path\to\malware.exe`
- Xóa scheduled task: `schtasks /delete /tn "TaskName" /f`
- Xóa service: `sc delete "ServiceName"`
- Xóa registry persistence: `reg delete "HKLM\Software\Microsoft\Windows\CurrentVersion\Run" /v "Malware" /f`

**2. Linux Endpoint:**
- Kill process: `kill -9 <pid>`
- Xóa file: `rm -rf /path/to/malware`
- Xóa crontab: `crontab -r`
- Xóa systemd service: `systemctl disable malicious.service`

**3. Network:**
- Xóa các rules NAT/port forwarding bất thường trên pfSense

### Phase 5: Recovery (Phục hồi)
- Restore hệ thống từ backup sạch
- Patch các lỗ hổng đã bị exploit
- Đổi password tất cả user
- Re-image máy bị compromise (nếu cần)

### Phase 6: Lessons Learned (Bài học kinh nghiệm)
- Viết báo cáo incident
- Cập nhật detection rules để phát hiện sớm hơn
- Đề xuất cải thiện security controls

## 2. Splunk Dashboards cho SOC L1
**Dashboard 1 - Authentication Monitoring:**
```sql
index=windows EventCode=4625 OR EventCode=4624
| stats count by EventCode, user, src_ip
| eval action=case(EventCode=4625, "Failed", EventCode=4624, "Success")
| chart count over user by action
```

**Dashboard 2 - Network Anomaly:**
```sql
index=network
| stats sum(bytes_out) as total_bytes by src_ip, dest_ip
| where total_bytes > 1000000
| table src_ip, dest_ip, total_bytes
```

**Dashboard 3 - Process Monitoring:**
```sql
index=windows EventCode=4688
| stats count by Process_Name, user
| sort - count
```

**Dashboard 4 - Threat Intelligence Alert:**
```sql
index=* (src_ip=*) OR (dest_ip=*)
| lookup threat_intel.csv ip as src_ip OUTPUTNEW threat
| where threat="malicious"
| table _time, src_ip, dest_ip, threat
```

## 3. Quy trình Escalation lên SOC L2
SOC L1 sẽ escalate lên SOC L2 khi:
- ❌ Không xác định được root cause
- ❌ Incident ảnh hưởng đến nhiều hệ thống (>3)
- ❌ Có dấu hiệu APT (Advanced Persistent Threat)
- ❌ Cần phân tích forensics chuyên sâu
- ❌ Cần sự cho phép để thực hiện containment actions

**Mẫu handover note:**
```
--- ESCALATION HANDOVER ---
Ticket ID: INC-2026-XXXX
Priority: High
Reported: [Thời gian]
Asset: [IP/Hostname]
Alert: [Loại alert]
Initial Findings:
- [Mô tả ngắn gọn]
Actions Taken:
- [Đã làm gì]
Reason for Escalation:
- [Lý do]
Attachment: [Logs/screenshots]
--- END ---
```

