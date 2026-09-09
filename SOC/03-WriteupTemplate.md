# WRITEUP: [Tên kịch bản]

## Mục tiêu
[Mục tiêu của kịch bản]

## Môi trường
- Attacker IP: 192.168.56.100
- Victim IP: 192.168.56.20
- Công cụ sử dụng: [Tool1, Tool2, ...]

## Các bước thực hiện

### Bước 1: [Tên bước]
```bash
# Command thực hiện
[command]
```
**Kết quả:** [Mô tả kết quả]

### Bước 2: [Tên bước]
...

## Phát hiện từ phía SOC

### Splunk Query
```sql
[Query phát hiện]
```

### Kết quả phân tích
[Phân tích kết quả]

## MITRE ATT&CK
| **Tactic** | **Technique** | **ID** |
| --- | --- | --- |
| [Tactic] | [Technique] | TXXXX |

## Bài học
[Bài học rút ra]


### 6.3. Các chỉ số cần đo lường (Metrics)
| Chỉ số | Mô tả | Mục tiêu |
|--------|-------|----------|
| **MTTD** | Mean Time to Detect | < 30 phút |
| **MTTR** | Mean Time to Respond | < 2 giờ |
| **Số lượng alert/ngày** | Volume alerts | Theo dõi trend |
| **False Positive Rate** | % alert sai | < 10% |
| **Tỷ lệ escalated** | % alert lên L2 | < 20% |

---
## Phụ lục: Tài liệu tham khảo

### Nội dung khóa BTL1 (6 domains)[reference:9][reference:10]

| Domain | Nội dung chính | Công cụ |
|--------|---------------|---------|
| **Security Fundamentals** | OSI model, security controls, networking | - |
| **Phishing Analysis** | Header fields, authentication checks, IOC types, reputation services, sandboxes | PhishTool, URL2PNG, CyberChef[reference:11] |
| **Threat Intelligence** | IOCs vs TTPs, Pyramid of Pain, ATT&CK framework | MISP, OpenCTI, DomainTools[reference:12] |
| **Digital Forensics** | Registry, Event IDs, Prefetch, Shimcache, Amcache, LNK, NTFS, /var/log | Autopsy, KAPE, FTK Imager, Volatility, Scalpel, ExifTool[reference:13] |
| **SIEM Analysis** | SPL basics, ECS field mapping, detection queries | Splunk, BOTS datasets[reference:14] |
| **Incident Response** | 6 phases, Windows/Linux triage commands | PowerShell, DeepBlueCLI[reference:15] |

### Các nguồn dữ liệu thực hành

- **BOTS (Boss of the SOC) datasets**: Dữ liệu Splunk mô phỏng[reference:16]
- **Malware Traffic Analysis**: PCAP mẫu để phân tích network
- **Phishing email samples**: Các mẫu email phishing để phân tích[reference:17]

### Công cụ chính trong BTL1[reference:18]

| Công cụ | Mục đích |
|---------|----------|
| **Splunk** | SIEM Analysis |
| **Autopsy** | Disk Forensics |
| **Volatility** | Memory Forensics |
| **Wireshark** | Network Analysis |
| **KAPE** | Artifact collection |
| **DeepBlueCLI** | PowerShell log analysis |
| **CyberChef** | Data decoding/encoding |
| **FTK Imager** | Disk imaging |
