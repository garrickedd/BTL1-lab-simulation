# BÁO CÁO ĐIỀU TRA SỰ CỐ BẢO MẬT

## 1. Thông tin chung
| Mục| Nội dung|
|-----|----------|
| **Số báo cáo**| IR-2026-XXXX|
| **Ngày báo cáo**| DD/MM/YYYY|
| **Người báo cáo**| [Tên SOC Analyst]|
| **Mức độ nghiêm trọng**| [Critical/High/Medium/Low]|
| **Trạng thái**| [Open/Investigating/Contained/Resolved/Closed]|

## 2. Tóm tắt sự cố
[Mô tả ngắn gọn sự cố: cái gì đã xảy ra, khi nào, ảnh hưởng đến ai]

## 3. Timeline sự kiện
| Thời gian| Sự kiện| Ghi chú|
|-----------|---------|---------|
| HH:MM DD/MM| [Sự kiện 1]| [Chi tiết]|
| HH:MM DD/MM| [Sự kiện 2]| [Chi tiết]|

## 4. Phân tích kỹ thuật

### 4.1. Indicator of Compromise (IoC)
| Loại| Giá trị| Nguồn phát hiện|
|------|---------|-----------------|
| IP| X.X.X.X| Splunk / Firewall|
| Domain| malicious.com| DNS log|
| File Hash| MD5/SHA256| Endpoint|
| Process| malware.exe| Event Log|

### 4.2. MITRE ATT&CK Mapping
| Tactic| Technique| ID| Observed|
|--------|-----------|-----|----------|
| [Tactic]| [Technique]| TXXXX| [Evidence]|

### 4.3. Splunk Queries sử dụng
```spl
[Query đã dùng để phát hiện]
```

## 5. Hành động đã thực hiện
- Cô lập asset
- Khóa tài khoản
- Xóa malware
- Block C2
- Khác: ...
    

## 6. Đề xuất khắc phục
- [Đề xuất 1]
- [Đề xuất 2]

## 7. Bài học kinh nghiệm
- [Bài học 1]
- [Bài học 2]

## 8. Phụ lục
- Logs
- Screenshots
- PCAP files
