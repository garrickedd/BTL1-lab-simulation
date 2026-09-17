# Hạ tầng phục vụ lab
## 1. Thiết kế hạ tầng
### 1.1. Tài nguyên hiện có
| Thiết bị    | OS    | CPU    | RAM    | GPU    |
| ------- | ------- | ------- | ------- | ------- |
| Desktop (HOST-A)    | Windows 11    | Ryzen 5 5600    | 16gb    | Rx570 4gb    |
| Laptop (HOST-B)    | Windows 11    | Ryzen 7 5800H    | 16gb    | (tích hợp)    |

### 1.2. Sơ đồ tổng quan
```
                    ┌─────────────────────┐
                    │  Internet (Modem)   │
                    │    192.168.1.1      │
                    └──────────┬──────────┘
                               │
              ┌────────────────┴────────────────┐
              │                                 │
    ┌─────────▼─────────┐             ┌─────────▼─────────┐
    │  HOST-A (Desktop) │             │  HOST-B (Laptop)  │
    │  LAN Cable        │             │  WiFi             │
    │  192.168.1.10     │             │  192.168.1.20     │
    └─────────┬─────────┘             └─────────┬─────────┘
              │                                 │
    ┌─────────▼────────────────────────────┐    │
    │  VM-pfSense (Host-A)                 │    │
    │  ├─ WAN (NAT): 10.0.2.15             │    │
    │  └─ LAN (Bridged): 192.168.56.1      │    │
    └─────────┬────────────────────────────┘    │
              │                                  │
    ┌─────────▼──────────────┐      ┌────────────▼─────────────────────────┐
    │  VMs trên Host-A       │      │  VMs trên Host-B                     │
    │                        │      │                                      │
    │  ┌──────────────────┐  │      │  ┌────────────────────────────────┐  │
    │  │ VM-SPLUNK        │  │      │  │ VM-ATTACKER                    │  │
    │  │ 192.168.56.10    │  │      │  │ ├─ Adapter 1: Bridged          │  │
    │  └──────────────────┘  │      │  │ │   IP: 192.168.56.100         │  │
    │  ┌──────────────────┐  │      │  │ │   GW: 192.168.56.1           │  │
    │  │ VM-WIN-ENDPOINT  │  │      │  │ └─ Adapter 2: NAT              │  │
    │  │ 192.168.56.20    │  │      │  │     IP: 10.0.2.15              │  │
    │  └──────────────────┘  │      │  │     GW: 10.0.2.2               │  │
    │  ┌──────────────────┐  │      │  └────────────────────────────────┘  │
    │  │ VM-LINUX-ENDPOINT│  │      │                                      │
    │  │ 192.168.56.30    │  │      │  ┌────────────────────────────────┐  │
    │  └──────────────────┘  │      │  │ VM-FORENSICS                   │  │
    │                        │      │  │ ├─ Adapter 1: Bridged          │  │
    │                        │      │  │ │   IP: 192.168.56.40          │  │
    │                        │      │  │ │   GW: 192.168.56.1           │  │
    │                        │      │  │ └─ Adapter 2: NAT              │  │
    │                        │      │  │     IP: 10.0.2.15              │  │
    │                        │      │  │     GW: 10.0.2.2               │  │
    │                        │      │  └────────────────────────────────┘  │
    └────────────────────────┘      └──────────────────────────────────────┘
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
#### 2.2.1. Thiết lập Host-Only Network trên VirtualBox
Trước khi tạo máy ảo, cần tạo sẵn card mạng ảo Host-Onlu để các VM giao tiếp với nhau và với máy thật
1. Mở VirtualBox -> File -> Host Network Manager
2. Nhấn Create (nếu chưa có). Mặc định VirtualBox sẽ tạo `vboxnet0`
3. Chọn `vboxnet0` và cấu hình:
- IPv4 Address: `192.168.56.1` (Đây là IP của máy host trên mạng Lab)
- IPv4 Network Mask: `255.255.255.0`
- DHCP Server: Disable (Tắt DHCP Server của VirtualBox). Lý do: Chúng ta sẽ cấp phát IP tĩnh thủ công hoặc để pfSense làm DHCP Server, tránh xung đột IP.

#### 2.2.2. Cấu hình Card Mạng cho từng VM trên host-A
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

#### 2.2.3. Setup rule pfSense (Interface LAN)**
Bộ rule theo thứ tự ưu tiên:
```
┌─────────────────────────────────────────────────────────────────────────────┐
│ LAN Rules (theo thứ tự từ trên xuống)                                        │
├────┬──────────┬──────────┬─────────────────────┬────────┬──────────────────┤
│ #  │ Action   │ Protocol │ Source              │ Dest   │ Description      │
├────┼──────────┼──────────┼─────────────────────┼────────┼──────────────────┤
│ 1  │ Pass     │ *        │ LAN Address         │*       │ Anti-Lockout     │
│ 2  │ Pass     │ IPv4 *   │ LAN subnets         │*       │ Allow LAN to Net │
│ 3  │ Pass     │ TCP/UDP  │ LAN subnets         │* :53   │ Allow DNS        │
│ 4  │ Pass     │ TCP      │ LAN subnets         │*:80,443│ Allow HTTP/S     │
│ 5  │ Block    │ IPv4 *   │ 192.168.56.100      │ LAN net│ Block Attacker   │
│ 6  │ Pass     │ IPv4 *   │ LAN net             │ LAN net│ Allow internal   │
└────┴──────────┴──────────┴─────────────────────┴────────┴──────────────────┘
```
**Rule #1: Anti-Lockout (Mặc định của pfSense)**
| Trường    | Giá trị    |
| ------- | ------- |
| Action    | Pass    |
| Interface    | LAN    |
| Protocol    | *    |
| Source    | LAN Address (192.168.56.1)    |
| Destination    | *    |
| Port    | 80, 443    |
>Mục đích: Cho phép truy cập Web GUI pfSense từ chính nó. Không được xóa.

**Rule #2: Allow LAN to Internet **
| Trường    | Giá trị    |
| ------- | ------- |
| Action    | Pass    |
| Interface    | LAN    |
| Protocol    | IPv4 *    |
| Source    | LAN subnets    |
| Destination    | *    |
| Port    | *    |
>Mục đích: Cho phép tất cả VM trong LAN (192.168.56.0/24) ra Internet qua pfSense (Splunk, Win-Endpoint, Linux-Endpoint).
>Đây là rule quan trọng nhất — nếu để LAN Address như ban đầu, các VM không ra được Internet.

**Rule #3: Allow DNS**
| Trường    | Giá trị    |
| ------- | ------- |
| Action    | Pass    |
| Interface    | LAN    |
| Protocol    | TCP/UDP    |
| Source    | LAN subnets    |
| Destination    | *    |
| Port    | 53    |
>Mục đích: Cho phép DNS query từ các VM (cần thiết nếu pfSense làm DNS Resolver).

>💡 Lưu ý: Rule này có thể không cần thiết nếu Rule #2 đã cho phép mọi traffic. Nhưng để rõ ràng, có thể thêm.

**Rule #4: Allow HTTP/HTTPS**
| Trường    | Giá trị    |
| ------- | ------- |
| Action    | Pass    |
| Interface    | LAN    |
| Protocol    | TCP    |
| Source    | LAN subnets    |
| Destination    | *    |
| Port    | 80, 443    |
>Mục đích: Cho phép duyệt web (HTTP/HTTPS). Cũng có thể bỏ nếu Rule #2 đã bao quát.

**Rule #5: Block VM-ATTACKER (Tùy chọn — Bật/tắt theo nhu cầu)**
| Trường    | Giá trị    |
| ------- | ------- |
| Action    | Block    |
| Interface    | LAN    |
| Protocol    | IPv4 *    |
| Source    | 192.168.56.100    |
| Destination    | LAN net    |
| Port    | *    |
| Description |	Block VM-ATTACKER to internal endpoints (enable or test) |
>Mục đích: Ngăn VM-ATTACKER tấn công các endpoint khi chưa thực hành.

>⚠️ QUAN TRỌNG: Rule này CHỈ có tác dụng khi traffic đi qua pfSense. Vì Attacker và Endpoint cùng subnet (192.168.56.0/24), traffic đi trực tiếp Layer 2 → rule này KHÔNG chặn được.
Nếu muốn chặn thực sự, phải tách subnet (OPT1) hoặc dùng cơ chế khác.
Trong lab: Rule này gần như vô dụng — có thể xóa hoặc giữ để làm ví dụ.

**Rule #6: Allow Internal Lab Communication**
| Trường    | Giá trị    |
| ------- | ------- |
| Action    | Pass    |
| Interface    | LAN    |
| Protocol    | IPv4 *    |
| Source    | LAN net    |
| Destination    | LAN net    |
| Port    | *    |
>Mục đích: Cho phép các VM trong LAN nói chuyện với nhau (nhưng thực tế chúng không qua pfSense).

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
#### 3.3.1. Cấu hình mạng
```bash
# 1. Kiểm tra interface
ip a
# Sẽ thấy: eth0 (Bridged) và eth1 (NAT)

# 2. Xem tên kết nối NetworkManager
nmcli connection show
```

**Cấu hình Adapter 1(Bridged):**
```bash
sudo nmcli connection modify "Wired connection 1" \
    ipv4.addresses 192.168.56.100/24 \
    ipv4.gateway 192.168.56.1 \
    ipv4.dns "1.1.1.1 8.8.8.8" \
    ipv4.method manual \
    ipv4.never-default yes \
    connection.autoconnect yes

sudo nmcli connection down "Wired connection 1"
sudo nmcli connection up "Wired connection 1"
```
>⚠️ Điểm mấu chốt: `ipv4.never-default yes` — nghĩa là KHÔNG dùng Bridged làm default route. Default route sẽ do NAT đảm nhiệm.

**Cấu hình Adapter 2 (NAT) - để kết nối internet:**
```bash
# Tạo kết nối mới cho eth1
sudo nmcli connection add type ethernet con-name "nat-internet" ifname eth1 \
    ipv4.method auto \
    ipv4.route-metric 100 \
    connection.autoconnect yes

sudo nmcli connection up "nat-internet"
```
>⚠️ Điểm mấu chốt: `ipv4.route-metric 100` — metric thấp hơn Bridged → NAT sẽ là default route.

**Giải thích cách hoạt động**
Khi Attacker tấn công Endpoint (192.168.56.20):
```
VM-ATTACKER (192.168.56.100)
    │
    ├─ Gói tin đến 192.168.56.20 (vm-win-endpoint)
    │
    ├─ Route lookup: 192.168.56.0/24 dev eth0 (cùng subnet)
    │
    ├─ Gửi trực tiếp qua eth0 (Bridged, Layer 2) → Endpoint
    │
    └─ KHÔNG qua pfSense → Nhanh, nhưng pfSense không log
```
Khi Attacker cần tải tool từ Internet:
```
VM-ATTACKER
    │
    ├─ Gói tin đến 8.8.8.8
    │
    ├─ Route lookup: default via 10.0.2.2 dev eth1
    │
    ├─ Gửi qua eth1 (NAT) → VirtualBox NAT → Modem → Internet
    │
    └─ KHÔNG qua pfSense → Nhanh hơn ✅
```

#### 3.3.2. Các công cụ cần cài đặt/sẵn có trên kali:
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
#### 3.4.1. Cấu hình mạng
**Cấu hình Adapter 1 (Bridged) - IP tĩnh**
```powershell
# Đặt IP tĩnh cho adapter Bridged
New-NetIPAddress -InterfaceAlias "Ethernet" `
    -IPAddress 192.168.56.40 `
    -PrefixLength 24 `
    -DefaultGateway 192.168.56.1

# Đặt DNS
Set-DnsClientServerAddress -InterfaceAlias "Ethernet" `
    -ServerAddresses 1.1.1.1, 8.8.8.8

# Kiểm tra
Get-NetIPAddress -InterfaceAlias "Ethernet"
```

**Cấu hình Adapter 2 (NAT) - DHCP**
```powershell
# Đặt DHCP cho adapter NAT
Set-NetIPInterface -InterfaceAlias "Ethernet 2" -Dhcp Enabled

# Renew DHCP
ipconfig /renew "Ethernet 2"

# Kiểm tra
Get-NetIPAddress -InterfaceAlias "Ethernet 2"
# Sẽ thấy IP 10.0.2.x
```

**Đặt metric để NAT là default route**
Đây là bước quan trọng nhất — để Internet đi qua NAT (nhanh) thay vì qua Bridged + pfSense:
```powershell
# Xem metric hiện tại
Get-NetIPInterface | Where-Object {$_.AddressFamily -eq "IPv4"} | Select InterfaceAlias, InterfaceMetric

# Đặt metric thấp cho NAT (ưu tiên cao)
Set-NetIPInterface -InterfaceAlias "Ethernet 2" -InterfaceMetric 10

# Đặt metric cao cho Bridged (ưu tiên thấp)
Set-NetIPInterface -InterfaceAlias "Ethernet" -InterfaceMetric 100

# Kiểm tra lại
Get-NetIPInterface | Where-Object {$_.AddressFamily -eq "IPv4"} | Select InterfaceAlias, InterfaceMetric
```
> 💡 Nguyên lý: Windows chọn default route dựa trên metric thấp nhất + route metric. Với NAT = 10 và Bridged = 100 → NAT sẽ là default route.

**Xóa default gateway của Bridged (nếu cần)**
Windows đôi khi vẫn dùng gateway của Bridged. Để chắc chắn, xóa default gateway của Bridged:
```powershell
# Xóa default gateway của Bridged (nhưng giữ IP)
Remove-NetRoute -InterfaceAlias "Ethernet" -DestinationPrefix "0.0.0.0/0" -Confirm:$false
```
>⚠️ Lưu ý: Vẫn giữ IP `192.168.56.40` và subnet route `192.168.56.0/24` để tấn công/forensics với các VM khác.

#### 3.4.2. Các công cụ cần cài:
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