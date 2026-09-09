# Kịch bản 2: Reconnaissance & Port Scanning
**Mục tiêu:** Attacker quét mạng nội bộ để phát hiện máy chủ và dịch vụ.

**Các bước thực hiện (trên VM-ATTACKER):**
``` bash
# 1. Ping sweep - phát hiện host sống
nmap -sn 192.168.56.0/24

# 2. Quét port nhanh
nmap -T4 -F 192.168.56.0/24

# 3. Quét chi tiết các dịch vụ
nmap -sV -sC -p- 192.168.56.10   # Splunk server
nmap -sV -sC -p- 192.168.56.20   # Windows endpoint

# 4. Quét SMB
nmap -p 445 --script smb-enum-shares,smb-os-discovery 192.168.56.20
```

**SOC phát hiện:**

- Splunk nhận log từ pfSense: Nhiều kết nối đến nhiều port trên cùng IP
- **Detection query**:

``` sql
index=network src_ip=192.168.56.100
| stats count by dest_ip, dest_port
| where count > 20
| table _time, src_ip, dest_ip, dest_port, count
```

