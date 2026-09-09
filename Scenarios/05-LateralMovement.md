# Kịch bản 5: Lateral Movement

**Mục tiêu:** Attacker di chuyển từ endpoint này sang endpoint khác.

**Thực hiện trên WM-ATTACKER (sau khi có shell trên VM-WIN-ENDPOINT):**

```bash
# Sử dụng PsExec để di chuyển sang máy khác
psexec \\192.168.56.30 -u administrator -p password cmd.exe

# Hoặc sử dụng WMI
wmic /node:192.168.56.30 /user:administrator /password:password process call create "cmd.exe"

# Sử dụng PowerShell Remoting
Enter-PSSession -ComputerName 192.168.56.30 -Credential administrator
```

**SOC phát hiện:**
- Windows Event IDs:
    - 4624: Logon type 3 (Network logon) từ IP lạ
    - 4688: Process creation với PsExec/WMIC
    - 5140: SMB share access
- Network: Kết nối SMB (445), RPC (135), WinRM (5985/5986)
- Splunk queries:
```sql
index=windows EventCode=4624 LogonType=3
| table _time, src_ip, user, dest_ip

index=windows EventCode=4688 Process_Name="psexec.exe" OR Process_Name="wmic.exe"
| table _time, user, Process_Name, CommandLine
```