# Asterisk 22 trên Debian – Hướng dẫn rút gọn (Checklist)

> Bản tóm tắt các bước **cài đặt, cấu hình và test gọi nội bộ**. Chỉ cần copy–paste theo thứ tự.

---

## 0) Chuẩn bị
- Debian 12/13 (VM hoặc máy vật lý), IP tĩnh trong LAN.
- Tối thiểu: 2 vCPU, 2GB RAM, 20GB disk.
- Mở cổng (nếu có firewall/router): **UDP 5060** và **UDP 10000–20000** (RTP).

---

## 1) Cài phụ thuộc & cập nhật
```bash
apt update && apt upgrade -y
apt install -y build-essential git wget curl subversion   libncurses5-dev libssl-dev libxml2-dev uuid-dev   libjansson-dev libsqlite3-dev libedit-dev
```
> Nếu dùng user thường: thêm `sudo` trước mỗi lệnh hoặc `su -` để vào root.

---

## 2) Tải & cài đặt Asterisk 22 (từ source)
```bash
cd /usr/src
wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-22-current.tar.gz
tar zxvf asterisk-22-current.tar.gz
cd asterisk-22.*
contrib/scripts/get_mp3_source.sh
contrib/scripts/install_prereq install

# Cấu hình build (đặt prefix về /usr để service tìm đúng binary)
./configure --prefix=/usr --sysconfdir=/etc --localstatedir=/var

# (tuỳ chọn) menuselect
# make menuselect

make -j"$(nproc)"
make install
make samples
make config
ldconfig
```

---

## 3) Tạo user & quyền chạy
```bash
adduser --system --group --home /var/lib/asterisk asterisk || true
mkdir -p /etc/asterisk /var/{lib,log,spool}/asterisk
chown -R asterisk:asterisk /etc/asterisk /var/{lib,log,spool}/asterisk
```

---

## 4) Khởi động dịch vụ
```bash
systemctl daemon-reload
systemctl enable asterisk
systemctl restart asterisk
systemctl status asterisk --no-pager
asterisk -rvvv -x "core show version"
```

---

## 5) Cấu hình **gọi nội bộ** (1001 ↔ 1002)

### 5.1 `pjsip.conf`
Sao lưu & ghi nội dung tối thiểu:
```bash
[ -f /etc/asterisk/pjsip.conf ] && cp /etc/asterisk/pjsip.conf /etc/asterisk/pjsip.conf.bak.$(date +%F-%H%M)

cat > /etc/asterisk/pjsip.conf <<'EOF'
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0
; Nếu đăng ký từ Internet/NAT, bỏ comment 3 dòng dưới và sửa đúng:
;external_media_address = <WAN_IP_or_FQDN>
;external_signaling_address = <WAN_IP_or_FQDN>
;local_net = 192.168.1.0/24

[1001]
type=endpoint
context=internal
disallow=all
allow=ulaw,alaw
auth=auth1001
aors=1001
[auth1001]
type=auth
auth_type=userpass
username=1001
password=123456
[1001]
type=aor
max_contacts=1

[1002]
type=endpoint
context=internal
disallow=all
allow=ulaw,alaw
auth=auth1002
aors=1002
[auth1002]
type=auth
auth_type=userpass
username=1002
password=123456
[1002]
type=aor
max_contacts=1
EOF
```

### 5.2 `extensions.conf`
```bash
[ -f /etc/asterisk/extensions.conf ] && cp /etc/asterisk/extensions.conf /etc/asterisk/extensions.conf.bak.$(date +%F-%H%M)

cat > /etc/asterisk/extensions.conf <<'EOF'
[internal]
exten => 1001,1,Dial(PJSIP/1001)
exten => 1002,1,Dial(PJSIP/1002)

; Echo test (kiểm tra âm thanh 2 chiều)
exten => *43,1,Answer()
 same => n,Echo()
 same => n,Hangup()
EOF
```

### 5.3 Nạp lại & kiểm tra
```bash
asterisk -rx "core reload"
asterisk -rx "pjsip reload"
asterisk -rx "dialplan reload"
asterisk -rx "pjsip show endpoints"
```

---

## 6) Đăng ký 2 softphone
- Trên **Zoiper/MicroSIP**:
  - **Account 1**: user `1001`, pass `123456`, server = IP Debian, transport = UDP.
  - **Account 2**: user `1002`, pass `123456`, server = IP Debian.
- Kiểm tra đăng ký:
```bash
asterisk -rx "pjsip show contacts"
```

---

## 7) Test cuộc gọi
- Gọi **1001 ↔ 1002**.
- Gọi **`*43`** (Echo) để kiểm tra âm thanh 2 chiều.
- Bật debug khi cần:
```bash
asterisk -rvvv
core set verbose 5
pjsip set logger on      # tắt: pjsip set logger off
rtp set debug on         # tắt: rtp set debug off
```

---

## 8) (Tuỳ chọn) Gọi ra/vào qua SIP Trunk – mẫu REGISTER
> Thay thế chỗ IN HOA đúng thông tin nhà mạng.

### 8.1 Thêm vào `pjsip.conf`
```ini
[siptrunk-auth]
type=auth
auth_type=userpass
username=TRUNK_USER
password=TRUNK_PASS

[siptrunk-aor]
type=aor
contact=sip:sip.provider.com

[siptrunk-out]
type=endpoint
transport=transport-udp
context=from-trunk
disallow=all
allow=ulaw,alaw
aors=siptrunk-aor
outbound_auth=siptrunk-auth
from_domain=sip.provider.com
from_user=YOUR_DID
dtmf_mode=rfc4733
direct_media=no
force_rport=yes
rewrite_contact=yes
rtp_symmetric=yes

[siptrunk-reg]
type=registration
transport=transport-udp
outbound_auth=siptrunk-auth
server_uri=sip:sip.provider.com
client_uri=sip:TRUNK_USER@sip.provider.com
contact_user=TRUNK_USER
retry_interval=60
expiration=3600
```

### 8.2 Dialplan gọi ra/vào
```ini
[internal]
; ... (giữ 1001/1002 + *43 ở trên)
exten => _0X.,1,NoOp(Outbound to ${EXTEN} via siptrunk-out)
 same => n,Set(CALLERID(num)=YOUR_DID)
 same => n,Dial(PJSIP/${EXTEN}@siptrunk-out,60)
 same => n,Hangup()

[from-trunk]
exten => YOUR_DID,1,Dial(PJSIP/1001,30)
 same => n,Hangup()
```

### 8.3 Nạp & kiểm tra trunk
```bash
asterisk -rx "pjsip reload"
asterisk -rx "pjsip show endpoint siptrunk-out"
asterisk -rx "pjsip show registrations"   # phải thấy Registered (nếu dùng REGISTER)
```

---

## 9) Firewall & NAT
- Debian (UFW, nếu bật):
```bash
ufw allow 5060/udp
ufw allow 10000:20000/udp
```
- Router: NAT/Forward **UDP 5060** và **UDP 10000–20000** → IP server Asterisk.
- Nếu Internet/NAT: đã cấu hình `external_*` + `local_net` trong `[transport-udp]`.

---

## 10) Mẹo xử lý nhanh
- **Không đăng ký được**: kiểm tra IP/Firewall; `pjsip show contacts`; bật `pjsip set logger on`.
- **403/401 khi gọi ra**: sai `TRUNK_USER/PASS`, `from_user/from_domain` hoặc nhà mạng chưa mở IP.
- **404 khi gọi ra**: pattern số/dạng E.164 chưa đúng.
- **Không có tiếng/1 chiều tiếng**: NAT/port; đảm bảo `rtp_symmetric=yes`, `force_rport=yes`, `rewrite_contact=yes` và port RTP mở.

---

**Hoàn tất.** Thực hiện xong các bước 1→7 là gọi nội bộ 1001 ↔ 1002 chạy ngay.
