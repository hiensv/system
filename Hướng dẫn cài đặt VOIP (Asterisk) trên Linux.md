# 📞 Hướng dẫn cài đặt VOIP (Asterisk) trên Linux

## 1. Chuẩn bị môi trường
- Máy chủ Ubuntu/Debian (có thể là VM trên Proxmox).
- Quyền `root` hoặc user có `sudo`.
- Cập nhật hệ thống:

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 2. Cài đặt các gói cần thiết
```bash
sudo apt install -y wget build-essential subversion   libjansson-dev libxml2-dev libsqlite3-dev uuid-dev   libncurses5-dev libssl-dev
```

---

## 3. Tải và cài đặt Asterisk
```bash
cd /usr/src
sudo wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz
sudo tar xvf asterisk-20-current.tar.gz
cd asterisk-20.*/
```

Cài các script hỗ trợ:
```bash
sudo contrib/scripts/get_mp3_source.sh
sudo contrib/scripts/install_prereq install
```

Biên dịch và cài đặt:
```bash
./configure
make menuselect    # (có thể chọn module cần thiết)
make
sudo make install
sudo make samples
sudo make config
sudo ldconfig
```

---

## 4. Khởi động dịch vụ Asterisk
```bash
sudo systemctl start asterisk
sudo systemctl enable asterisk
```

Kiểm tra:
```bash
sudo asterisk -rvv
```

---

## 5. Cấu hình SIP extensions
Mở file cấu hình:
```bash
sudo nano /etc/asterisk/sip.conf
```

Thêm cấu hình:
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

---

## 6. Cấu hình Dialplan
Mở file:
```bash
sudo nano /etc/asterisk/extensions.conf
```

Thêm cấu hình:
```ini
[internal]
exten => 1001,1,Dial(SIP/1001)
exten => 1002,1,Dial(SIP/1002)
```

---

## 7. Kiểm tra và reload Asterisk
```bash
sudo asterisk -rvv
sip reload
dialplan reload
```

Xem danh sách SIP peers:
```bash
sip show peers
```

---

## 8. Kết nối Softphone
- Dùng phần mềm như **Zoiper** hoặc **Linphone**.
- Cấu hình:
  - Username: `1001` (hoặc `1002`)
  - Password: `1234` (hoặc `5678`)
  - Server: IP máy chủ Asterisk

---

## 9. Gọi thử
- Đăng nhập 2 softphone khác nhau (1001 và 1002).
- Thực hiện cuộc gọi thử giữa 2 extension.

---

✅ Vậy là bạn đã có hệ thống VOIP cơ bản với Asterisk.
