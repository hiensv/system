# 📞 Hướng dẫn cài đặt VOIP: Asterisk & FreePBX

---

## 🔹 Phần 1: Cài đặt **Asterisk (thuần)**

### 1. Chuẩn bị môi trường
- Hệ điều hành: Ubuntu 20.04/22.04 (hoặc Debian 11).  
- Update hệ thống:
```bash
sudo apt update && sudo apt upgrade -y
```

### 2. Cài các gói cần thiết
```bash
sudo apt install -y wget build-essential subversion   libjansson-dev libxml2-dev libsqlite3-dev uuid-dev   libncurses5-dev libssl-dev
```

### 3. Tải & cài Asterisk
```bash
cd /usr/src
sudo wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz
sudo tar xvf asterisk-20-current.tar.gz
cd asterisk-20.*/
```

Cài script hỗ trợ:
```bash
sudo contrib/scripts/get_mp3_source.sh
sudo contrib/scripts/install_prereq install
```

Build & cài đặt:
```bash
./configure
make menuselect    # chọn module
make
sudo make install
sudo make samples
sudo make config
sudo ldconfig
```

### 4. Khởi động Asterisk
```bash
sudo systemctl start asterisk
sudo systemctl enable asterisk
```

Vào CLI:
```bash
sudo asterisk -rvv
```

### 5. Cấu hình extensions SIP
File: `/etc/asterisk/sip.conf`
```ini
[general]
context=default
allowguest=no
srvlookup=yes
udpbindaddr=0.0.0.0

[1001]
type=friend
secret=1234
host=dynamic
context=internal

[1002]
type=friend
secret=5678
host=dynamic
context=internal
```

### 6. Cấu hình dialplan
File: `/etc/asterisk/extensions.conf`
```ini
[internal]
exten => 1001,1,Dial(SIP/1001)
exten => 1002,1,Dial(SIP/1002)
```

### 7. Reload Asterisk
```bash
sudo asterisk -rvv
sip reload
dialplan reload
```

Kiểm tra:
```bash
sip show peers
```

✅ Giờ bạn có thể dùng softphone (Zoiper, Linphone) để đăng nhập extension 1001 & 1002, gọi thử.

---

## 🔹 Phần 2: Cài đặt **FreePBX (dễ dùng)**

### 1. Chuẩn bị
- VM/Server mới trên **Proxmox**.  
- Tải ISO FreePBX Distro: [https://www.freepbx.org/downloads/](https://www.freepbx.org/downloads/)  
  (bao gồm CentOS + Asterisk + FreePBX).  

### 2. Cài đặt từ ISO
- Upload ISO vào Proxmox.  
- Tạo VM mới → chọn ISO FreePBX.  
- Boot và làm theo hướng dẫn cài đặt (Next → Next).  
- Sau khi cài xong, máy sẽ khởi động vào FreePBX OS.

### 3. Đăng nhập
- Đăng nhập console bằng user `root` (password đặt lúc cài).  
- Gõ `ifconfig` hoặc `ip addr` để lấy địa chỉ IP.  

### 4. Truy cập Web GUI
- Trên trình duyệt: `http://<IP-server>`  
- Đăng nhập và chạy **FreePBX initial setup wizard**.  
- Đặt mật khẩu quản trị web.  

### 5. Tạo Extensions
- Vào menu **Applications → Extensions → Add Extension**.  
- Chọn `Chan_SIP` hoặc `PJSIP` → nhập số (ví dụ 1001, 1002) → đặt password.  

### 6. Cấu hình Trunk & Outbound (nếu cần gọi ra PSTN)
- Vào **Connectivity → Trunks** để thêm SIP trunk.  
- Vào **Connectivity → Outbound Routes** để định tuyến cuộc gọi.  

### 7. Softphone đăng nhập
- Cài Zoiper hoặc Linphone trên máy.  
- Nhập thông tin extension (1001/1002) và IP server FreePBX.  

### 8. Thực hiện cuộc gọi thử
- Gọi từ extension 1001 → 1002.  
- Nếu kết nối thành công, bạn đã có hệ thống VOIP hoàn chỉnh.

---

## 📊 So sánh khi triển khai thực tế

| Tiêu chí       | **Asterisk** (thuần) | **FreePBX** (GUI) |
|----------------|----------------------|-------------------|
| Cài đặt        | Biên dịch thủ công   | ISO boot → Next   |
| Quản lý        | CLI + file config    | Web GUI trực quan |
| Mức độ dễ dùng | Khó (cho sysadmin)   | Dễ (cho IT support) |
| Linh hoạt      | Tùy biến vô hạn      | Theo module sẵn có |

---

👉 Nếu bạn triển khai lab học **kỹ thuật sâu** → chọn **Asterisk**.  
👉 Nếu bạn triển khai **doanh nghiệp thực tế** → chọn **FreePBX** cho nhanh. 
