# Hạ tầng phục vụ lab
## 1. Thiết kế hạ tầng
### 1.1. Tài nguyên hiện có
| Thiết bị    | OS    | CPU    | RAM    | GPU    |
| ------- | ------- | ------- | ------- | ------- |
| Desktop (HOST-A)    | Windows 11    | Ryzen 5 5600    | 16gb    | Rx570 4gb    |
| Laptop (HOST-B)    | Windows 11    | Ryzen 7 5800H    | 16gb    | (tích hợp)    |

### 1.2. Sơ đồ tổng quan
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              MẠNG LAB SOC                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────────────────────┐          ┌───────────────────────────┐       │
│  │      HOST-A (Desktop)     │          │      HOST-B (Laptop)      │       │
│  │    Ryzen 5 5600 - 16GB    │          │   Ryzen 7 5800H - 16GB    │       │
│  │                           │          │                           │       │
│  │  ┌─────────────────────┐  │          │  ┌─────────────────────┐  │       │
│  │  │  VM-SPLUNK          │  │          │  │  VM-ATTACKER        │  │       │
│  │  │  (Splunk Enterprise)│  │          │  │  (Kali Linux)       │  │       │
│  │  │  Ubuntu Server      │  │          │  │  + công cụ tấn công │  │       │
│  │  └──────────┬──────────┘  │          │  └──────────┬──────────┘  │       │
│  │             │             │          │             │             │       │
│  │  ┌──────────▼──────────┐  │          │  ┌──────────▼──────────┐  │       │
│  │  │  VM-WIN-ENDPOINT    │  │          │  │  VM-FORENSICS       │  │       │
│  │  │  (Windows 10/11)    │  │          │  │  (Windows 10)       │  │       │
│  │  │  + Splunk UF        │  │          │  │  + Autopsy/KAPE     │  │       │
│  │  └──────────┬──────────┘  │          │  └─────────────────────┘  │       │
│  │             │             │          │                           │       │
│  │  ┌──────────▼──────────┐  │          │                           │       │
│  │  │  VM-LINUX-ENDPOINT  │  │          │                           │       │
│  │  │  (Ubuntu Server)    │  │          │                           │       │
│  │  │  + Splunk UF        │  │          │                           │       │
│  │  └─────────────────────┘  │          │                           │       │
│  └───────────────────────────┘          └───────────────────────────┘       │
│                                                                             │
│  ═══════════════════════════════════════════════════════════════════════    │
│                    Mạng nội bộ Lab (NAT Network / Host-Only)                │
│                             192.168.56.0/24                                 │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.3. Mô hình kết nối chi tiết
```
                      ┌─────────────────────┐
                      │   Internet (Host)   │
                      └──────────┬──────────┘
                                 │
                      ┌──────────▼──────────┐
                      │  pfSense Firewall   │ ◄─── Tường lửa ảo
                      │   (VM trên Host-A)  │       (NAT + Rules)
                      └──────────┬──────────┘
                                 │
                     ┌───────────▼───────────┐
                     │   Lab Internal Network│
                     │    192.168.56.0/24    │
                     └───────────┬───────────┘
                                 │
        ┌────────────────────────┼────────────────────────┐
        │                        │                        │
┌───────▼───────┐       ┌───────▼────────┐       ┌───────▼───────┐
│  VM-SPLUNK    │       │ VM-WIN-ENDPOINT│       │ VM-ATTACKER   │
│  10.0.0.10/24 │       │  10.0.0.20/24  │       │  10.0.0.100/24│
│  (Splunk Srv) │       │  (Windows)     │       │  (Kali)       │
└───────────────┘       └────────────────┘       └───────────────┘
        │                        │                        │
┌───────▼───────┐       ┌───────▼───────┐
│ VM-LINUX-ENDP │       │ VM-FORENSICS  │
│  10.0.0.30/24 │       │  10.0.0.40/24 │
│  (Ubuntu)     │       │  (Windows)    │
└───────────────┘       └───────────────┘
```

## 2. Chi tiết hạ tầng
### 2.1. Phân bổ tài nguyên
#### Trên HOST-A
| VM    | OS    | RAM    | CPU    | Disk    | Vai trò    |
| ------- | ------- | ------- | ------- | ------- | ------- |
| VM-pfSense    | pfSense CE    | 1GB    | 1vCPU    | 10GB    | Tường lửa/Router    |
| VM-SPLUNK    | Ubuntu Server 24.04    | 6GB    | 2vCPU    | 50GB    | Splunk Enterprise Server    |
| VM-WIN-ENDPOINT    | Windows 10 ltsc 2021    | 4GB    | 2vCPU    | 40GB    | Endpoint Windows + Splunk UF    |
| VM-LINUX-ENDPOINT    | Ubuntu Server 24.04    | 2GB    | 1vCPU    | 20GB    | Endpoint Linux + Splunk UF    |
| Tổng    |     | 13GB    | 6vCPU    | 120GB    |     |

#### Trên HOST-B
| VM    | OS    | RAM    | CPU    | Disk    | Vai trò    |
| ------- | ------- | ------- | ------- | ------- | ------- |
| VM-ATTACKER    | Kali Linux    | 4GB    | 2vCPU    | 40GB    | Attacker tools    |
| VM-FORENSICS    | Windows 10 ltsc 2021    | 4GB    | 2vCPU    | 40GB    | Forensics tools (Autopsy, KAPE, FTK Imager)    |
| Tổng    |     | 8GB    | 4vCPU    | 80GB    |     |

### 2.2. Cấu hình network
#### Thiết lập Host-Only Network trên VirtualBox
Trước khi tạo máy ảo, cần tạo sẵn card mạng ảo Host-Onlu để các VM giao tiếp với nhau và với máy thật
1. Mở VirtualBox -> File -> Host Network Manager
2. Nhấn Create (nếu chưa có). Mặc định VirtualBox sẽ tạo `vboxnet0`
3. Chọn `vboxnet0` và cấu hình:
- IPv4 Address: `192.168.56.1` (Đây là IP của máy host trên mạng Lab)
- IPv4 Network Mask: `255.255.255.0`
- DHCP Server: Disable (Tắt DHCP Server của VirtualBox). Lý do: Chúng ta sẽ cấp phát IP tĩnh thủ công hoặc để pfSense làm DHCP Server, tránh xung đột IP.

#### Cấu hình Card Mạng cho từng V
**A. VM-pfSense (Cần 2 card mạng)**
| Adapter    | Loại (Attached to)    | Tên (Name)    | Mục đích    | Ghi chú    |
| ------- | ------- | ------- | ------- | ------- |
| Adapter1    | NAT    | (Mặc định)    | WAN Port    | Kết nối ra internet thông qua máy host. VirtualBox sẽ cấp DHCP dải `10.0.2.0/24` cho adapter này    |
| Adapter2    | Host-Only Adapter    | `vboxnet0`    | LAN Port    | Kết nối vào mạng nội bộ Lab `192.168.56.0/24`    |

>  ⚠️ Lưu ý khi cài pfSense: Trong quá trình cài đặt, hãy gán WAN là em0 (Adapter 1) và LAN là em1 (Adapter 2). Sau khi cài xong, set IP tĩnh cho LAN là 192.168.56.1

**B. Các máy ảo còn lại**
Tất cả các máy này chỉ cần 1 card mạng duy nhất:
| Adapter    | Loại (Attached to)    | Tên (Name)    | Mục đích    |
| ------- | ------- | ------- | ------- |
| Adapter1    | Host-Only Adapter    | `vboxnet0`    | Kết nối trực tiếp vào mạng Lab `192.168.56.0/24`    |

Cấu hình IP tĩnh:
| Máy ảo    | Interface    | IP    | Gateway    | DNS    |
| ------- | ------- | ------- | ------- | ------- |
| pfSense (WAN)    | Bridge    | DHCP    | -    | 8.8.8.8    |
| pfSense (LAN)    | Host-Only    | 192.168.56.1    | -    | -    |
| VM-SPLUNK    | Host-Only    | 192.168.56.10    | 192.168.56.1    | 192.168.56.1    |
| VM-WIN-ENDPOINT    | Host-Only    | 192.168.56.20    | 192.168.56.1    | 192.168.56.1    |
| VM-LINUX-ENDPOINT    | Host-Only    | 192.168.56.30    | 192.168.56.1    | 192.168.56.1    |
| VM-FORENSICS    | Host-Only    | 192.168.56.40    | 192.168.56.1    | 192.168.56.1    |
| VM-ATTACKER    | Host-Only    | 192.168.56.100    | 192.168.56.1    | 192.168.56.1    |

**Setup rule pfSense**
```
┌─────────────────────────────────────────────────────────────┐
│                    pfSense Firewall Rules                   │
├─────────────────────────────────────────────────────────────┤
│ LAN → ANY:  Allow  (cho phép giao tiếp nội bộ)              │
│ LAN → WAN:  Allow  (cho phép truy cập Internet)             │
│ WAN → LAN:  Block  (chặn truy cập từ ngoài vào)             │
│ ATTACKER → ENDPOINTS: Block (mặc định, bật khi test)        │
└─────────────────────────────────────────────────────────────┘
```
1. Truy cập Web GUI -> LAN tab
2. Mặc định, có thể có rule kiểu "Default allow LAN to any". Rule này rất rộng. Đối với môi trường lab, tốt nhất là tạo các rule cụ thể hơn. Có thể disable hoặc xóa rule mặc định
3. Thêm rule theo thứ tự:
- **Rule 1: Allow all internal Lab traffic.**
    - **Action:** `Pass`
    - **Protocol:** `Any`
    - **Source:** `LAN net` (This represents 192.168.56.0/24)
    - **Destination:** `LAN net`
    - **Description:** `Allow internal lab communication`
- **Rule 2: Allow internal traffic out to the Internet.**
    - **Action:** `Pass`
    - **Protocol:** `Any` (or be more restrictive by allowing only `TCP/UDP` on ports like `80`, `443`, and `53` for a more secure lab)
    - **Source:** `LAN net`
    - **Destination:** `Any`
    - **Description:** `Allow LAN to Internet`
- **Rule 3: Block the Attacker VM from communicating with endpoints (controllable for testing).**
    - **Action:** `Block`
    - **Protocol:** `Any`
    - **Source:** Type `192.168.56.100/32` (the specific IP of the attacker)
    - **Destination:** `LAN net`
    - **Description:** `Block VM-ATTACKER to internal endpoints (enable for test)`
> *Tip:* To easily toggle this block rule on and off during testing, you can **check the `Disabled` box** for this rule when you create it. Then, you can simply check/uncheck it on the main rules page to enable or disable the block .

**Cách cấu hình ip trên Ubuntu Server**
1. Mở terminal, dùng lệnh `ip addr` để xác định tên giao diện mạng (ví dụ: `ens33` hoặc `enp0s3`).
2. Mở file cấu hình Netplan: `sudo nano /etc/netplan/01-netcfg.yaml` (tên file có thể khác, bạn dùng `ls /etc/netplan/` để xem).
3. Thêm cấu hình cho giao diện của bạn:
```
network:
  version: 2
  renderer: networkd
  ethernets:
    enp0s3:  # Thay bằng tên giao diện của bạn
      dhcp4: no
      addresses:
        - 192.168.56.10/24  # IP bạn muốn đặt
      routes:
        - to: default
          via: 192.168.56.1   # Gateway là pfSense
      nameservers:
        addresses: [192.168.56.1, 8.8.8.8]
```
4. Lưu file và áp dụng cấu hình: `sudo netplan apply`.

## Thiết lập SOC và Attacker
### 3.1. Splunk Server (VM-SPLUNK - Ubuntu Server 24.04)
**Cài đặt Splunk Enterprise**
```
# Tải Splunk Enterprise (bản free 500MB/ngày)
wget -O splunk-9.x.x-xxxxxx-linux-2.6-amd64.deb \
  "https://download.splunk.com/products/splunk/releases/9.x.x/linux/splunk-9.x.x-xxxxxx-linux-2.6-amd64.deb"
  
  

sudo dpkg -i splunk-9.x.x-xxxxxx-linux-2.6-amd64.deb

# Khởi động Splunk
sudo /opt/splunk/bin/splunk start --accept-license --run-as-root

# Enable boot-start
sudo /opt/splunk/bin/splunk enable boot-start

# Mở port management (8089) và web UI (8000)
sudo ufw allow 8000/tcp
sudo ufw allow 8089/tcp
sudo ufw allow 9997/tcp   # Port nhận dữ liệu từ forwarder
```

Truy cập Splunk Web: `http://192.168.56.10:8000`
Cấu hình receiver (nhận log từ forwarder):
```
Settings → Forwarding and Receiving → Configure receiving → Add new
Port: 9997
```

### 3.2. Splunk Universal Forwarder (trên Endpoints)
**Windows Endpoint (VM-WIN-ENDPOINT)**
```
# Tải Splunk Universal Forwarder cho Windows
# Cài đặt và cấu hình:
C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe add forward-server 192.168.56.10:9997
C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe enable boot-start
C:\Program Files\SplunkUniversalForwarder\bin\splunk.exe start
```

**Monitor các log Windows quan trọng:**
- Security Event Log (Event ID 4624, 4625, 4672, 4688, 4698, 4732, 4733...)
- System Event Log
- Application Event Log
- PowerShell Operational Log
- Sysmon logs (nếu cài Sysmon)

**Linux Endpoint (VM-LINUX-ENDPOINT):**
```
# Cài đặt Splunk UF
wget -O splunkforwarder-9.x.x-xxxxxx-linux-2.6-amd64.deb \
  "https://download.splunk.com/products/universalforwarder/releases/9.x.x/linux/splunkforwarder-9.x.x-xxxxxx-linux-2.6-amd64.deb"
sudo dpkg -i splunkforwarder-9.x.x-xxxxxx-linux-2.6-amd64.deb

# Cấu hình forward server
sudo /opt/splunkforwarder/bin/splunk add forward-server 192.168.56.10:9997

# Monitor logs
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/auth.log
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/syslog
sudo /opt/splunkforwarder/bin/splunk add monitor /var/log/ufw.log
```

### 3.3. Attacker Machine (VM-ATTACKER - Kali Linux)
**Các công cụ cần cài đặt/sẵn có trên kali:**
| Nhóm công cụ    | Công cụ cụ thể    | Mục đích    |
| ------- | ------- | ------- |
| Recon    | Nmap, Masscan, RustScan    | Quét mạng, phát hiện dịch vụ    |
| Web    | BurpSuite, Nikto, Gobusster, FFUF    | Kiểm thử web, fuzzing    |
| Exploit    | Metasploit, SearchSploit    | Khai thác lỗ hổng    |
| Password    | Hydra, John the Ripper, Hashcat    | Tấn công password    |
| Phishing    | GoPhish, Evilginx2 [bổ sung sau]    | Tạo chiến dịch phishing    |
| C2    | Covalt Strike (demo), Covenant, Silver    | Command & Control    |
| Persistence    | Empire, PowerShell Empire    | Persistence trên Windows    |
| Network    | Wireshark, tcpdump, responder    | Phân tích/đánh hơi netwwork    |
| PrivEsc    | WinPEAS, LinPEAS, BloodHound    | Leo thang đặc quyền    |

**Cài đặt thêm tools:**
```
# Cập nhật và cài đặt
sudo apt update && sudo apt upgrade -y
sudo apt install -y gobuster ffuf nikto hydra john hashcat bloodhound responder

# Cài GoPhish
wget https://github.com/gophish/gophish/releases/latest/download/gophish-vX.X.X-linux-64bit.zip
unzip gophish-*.zip -d gophish
cd gophish && ./gophish

# Cài Sliver C2
curl https://sliver.sh/install | sudo bash
```

### 3.4. Forensics Machine (VM-FORENSICS - Windows 10)
Các công cụ cần cài:
| Công cụ    | Mục đích    | Nguồn    |
| ------- | ------- | ------- |
| Autopsy / The Sleuth Kit    | Phân tích disk forensics    | autopsy.com    |
| FTK Imager    | Tạo image ổ đĩa, mount evidence    | AccessData    |
| KAPE (Kroll Artifact Parser)    | Thu thập artifacts nhanh    | Kroll    |
| Volitality 3    | Phân tích memory forensics    | github    |
| PECmd    | Phân tích Prefetch files    | Eric Zimmerman    |
| RBCmd    | Phân tích Recycler    | Eric Zimmerman    |
| MFTECmd    | Phân tích $MFT    | Eric Zimmerman    |
| JLECmd    | Phân tích Jump Lists    | Eric Zimmerman    |
| DeepBlueCLI    | Phân tích log PowerShell/Security    | Github    |
| CyberChef   | Xử lý dữ liệu mã hóa/giải mã   | github   |
| Wireshark     | Phân tích PCAP      | wireshark.org       |
| PhishTool     | Phân tích email phishing [Bổ sung sau]        | -

**Cài đặt Python và các dependencies cho Volitality**
```
# Cài Python 3.x từ python.org
# Cài Volatility 3
git clone https://github.com/volatilityfoundation/volatility3.git
cd volatility3
python -m pip install -r requirements.txt
```