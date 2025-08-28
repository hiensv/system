# 📞 VOIP Asterisk 20 trên Ubuntu 22.04 — Tài liệu tổng hợp & đúc rút triển khai thực tế

> Mục tiêu: dựng Asterisk 20, tạo 2 máy nhánh (1001/1002) gọi **nội bộ**, sau đó cấu hình **gọi ra ngoài** bằng SIP Trunk. Tài liệu dạng “SOP thợ” — ngắn gọn, đủ lệnh, có bước kiểm tra và xử lý lỗi.

---

## 0) Bối cảnh nhanh
- **Server Asterisk**: Ubuntu 22.04 (ví dụ IP LAN `172.16.95.10`).
- **Port cần mở**: SIP UDP `5060`, RTP UDP `10000–20000`.
- **Nếu ra Internet**: Port-forward từ router public IP → server; cấu hình NAT trong `pjsip.conf`.

---

## 1) Chuẩn bị hệ thống
```bash
apt update && apt upgrade -y
apt install -y wget curl build-essential subversion git   libjansson-dev libxml2-dev libsqlite3-dev uuid-dev   libncurses5-dev libssl-dev libedit-dev pkg-config

# Kiểm tra kết nối & DNS
ping -c 2 8.8.8.8
ping -c 2 google.com
```
> Nếu DNS lỗi, thêm DNS 8.8.8.8/1.1.1.1 vào netplan rồi `netplan apply`.

---

## 2) Cài đặt Asterisk 20 (build từ source)
```bash
cd /usr/src
wget https://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz  || wget http://downloads.asterisk.org/pub/telephony/asterisk/asterisk-20-current.tar.gz

tar xvf asterisk-20-current.tar.gz
cd asterisk-20.*/
contrib/scripts/get_mp3_source.sh
contrib/scripts/install_prereq install

./configure
make -j"$(nproc)"
make install
make samples
make config
ldconfig

systemctl enable asterisk
systemctl start asterisk
asterisk -rvvv   # vào CLI thử
exit
```

---

## 3) Cấu hình **PJSIP** — tạo 2 máy nhánh 1001/1002
File: `/etc/asterisk/pjsip.conf`
```ini
[transport-udp]
type=transport
protocol=udp
bind=0.0.0.0
; --- NAT (bật nếu có public IP/NAT) ---
;external_media_address=YOUR.PUBLIC.IP
;external_signaling_address=YOUR.PUBLIC.IP
;local_net=172.16.0.0/12

; ===== 1001 =====
[1001]
type=endpoint
context=internal
disallow=all
allow=ulaw,alaw
auth=auth1001
aors=1001
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[auth1001]
type=auth
auth_type=userpass
password=1234
username=1001

[1001]
type=aor
max_contacts=1

; ===== 1002 =====
[1002]
type=endpoint
context=internal
disallow=all
allow=ulaw,alaw
auth=auth1002
aors=1002
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[auth1002]
type=auth
auth_type=userpass
password=5678
username=1002

[1002]
type=aor
max_contacts=1
```

### Dialplan nội bộ — `/etc/asterisk/extensions.conf`
```ini
[internal]
exten => 1001,1,NoOp(Call to 1001)
 same => n,Dial(PJSIP/1001,30)
 same => n,Hangup()

exten => 1002,1,NoOp(Call to 1002)
 same => n,Dial(PJSIP/1002,30)
 same => n,Hangup()
```

Áp dụng & kiểm tra:
```bash
systemctl restart asterisk
asterisk -rx "pjsip show endpoints"
```

---

## 4) Firewall (nếu dùng UFW)
```bash
ufw allow 5060/udp
ufw allow 10000:20000/udp
ufw reload
```

---

## 5) Đăng ký thiết bị & gọi nội bộ
- **1001**: user `1001`, pass `1234`, server `172.16.95.10`
- **1002**: user `1002`, pass `5678`, server `172.16.95.10`

Kiểm tra đăng ký:
```bash
asterisk -rx "pjsip show contacts"
```
Gọi **1001 ↔ 1002**, theo dõi CLI:
```bash
asterisk -rvvv
core show channels
```

---

## 6) Gọi ra ngoài bằng **SIP Trunk**
> Lấy thông số từ nhà mạng (ITSP): `username`, `password`, `SIP domain`, **DID**.

### 6.1 PJSIP cho **trunk đăng ký (registration)**
Thêm cuối `/etc/asterisk/pjsip.conf` (đổi IN HOA theo thực tế):
```ini
[trunk-itsp]
type=registration
outbound_auth=trunk-itsp-auth
server_uri=sip:SIP.PROVIDER.COM
client_uri=sip:YOUR_USER@SIP.PROVIDER.COM
contact_user=YOUR_USER
retry_interval=60

[trunk-itsp-auth]
type=auth
auth_type=userpass
username=YOUR_USER
password=YOUR_PASS

[trunk-itsp-aor]
type=aor
contact=sip:SIP.PROVIDER.COM

[trunk-itsp-endpoint]
type=endpoint
transport=transport-udp
context=inbound-trunk
disallow=all
allow=ulaw,alaw
aors=trunk-itsp-aor
outbound_auth=trunk-itsp-auth
from_domain=SIP.PROVIDER.COM
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes
```

### 6.2 Nếu **IP-auth** (không đăng ký)
```ini
[trunk-itsp-aor]
type=aor
contact=sip:ITSP_IP_OR_DOMAIN

[trunk-itsp-endpoint]
type=endpoint
transport=transport-udp
context=inbound-trunk
disallow=all
allow=ulaw,alaw
aors=trunk-itsp-aor
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[trunk-itsp-identify]
type=identify
endpoint=trunk-itsp-endpoint
match=ITSP_IP_OR_SUBNET
```

### 6.3 Dialplan gọi ra/vào — `/etc/asterisk/extensions.conf`
```ini
[internal]
; ... (1001/1002 giữ nguyên)
include => outbound

[outbound]
; Bấm 0 trước số đích để gọi ra ngoài
exten => _0X.,1,NoOp(Outbound via ITSP)
 same => n,Dial(PJSIP/${EXTEN:1}@trunk-itsp,60)
 same => n,Hangup()

[inbound-trunk]
; ITSP đổ cuộc gọi vào đây -> chuyển đến 1001 (đổi theo nhu cầu)
exten => s,1,NoOp(Incoming from ITSP)
 same => n,Goto(internal,1001,1)
```
Nạp lại & kiểm tra:
```bash
asterisk -rx "core reload"
asterisk -rx "pjsip show registrations"   # nếu dùng registration
```

**Gọi ra**: trên 1001 bấm `0<số-đích>` (vd `0903xxxxxx`).  
**Gọi vào**: gọi DID → Asterisk đổ vào 1001.

---

## 7) NAT & lỗi âm thanh một chiều
- Forward UDP **5060** & **10000–20000** từ router → server.
- Trong `pjsip.conf` bật 3 dòng NAT (external_* & local_net).
- Bật `rtp_symmetric/force_rport/rewrite_contact` cho endpoint/trunk.
- Debug:
```bash
asterisk -rvvv
pjsip set logger on
rtp set debug on
```
Tắt: `pjsip set logger off`, `rtp set debug off`.

---

## 8) Bảo mật & vận hành tối thiểu
```bash
apt install -y fail2ban
cat >/etc/fail2ban/jail.d/asterisk.local <<'EOF'
[asterisk]
enabled = true
port    = 5060,5061
protocol = udp
filter  = asterisk
logpath = /var/log/asterisk/messages
maxretry = 6
bantime  = 3600
EOF
systemctl restart fail2ban
```
- Dùng mật khẩu mạnh cho extensions.
- Hạn chế pattern gọi quốc tế nếu chưa cần (`_00X.`).
- Sao lưu nhanh:
```bash
tar czf /root/asterisk-backup-$(date +%F).tgz   /etc/asterisk /var/lib/asterisk/agi-bin /var/spool/asterisk/voicemail
```

---

## 9) Checklist bàn giao ngắn gọn
- [ ] 1001/1002 đăng ký OK (`pjsip show contacts`).
- [ ] Gọi nội bộ 2 chiều OK, không mất tiếng.
- [ ] Trunk Registered/IP-auth OK.
- [ ] Gọi ra (prefix `0`) và gọi vào (DID) hoạt động.
- [ ] Đã bật Fail2Ban, mật khẩu mạnh.
- [ ] Đã mở port RTP & cấu hình NAT (nếu Internet).
- [ ] Đã có bản sao lưu cấu hình.

---

## 10) One-liner script (cài & tạo 1001/1002)
Có thể dùng script tự động hoá mình đính kèm:
```bash
bash install_asterisk_basic.sh
```
> Script tạo sẵn 1001/1002 (ULAW/ALAW), khởi động dịch vụ và in lệnh kiểm tra.

Chúc bạn dựng tổng đài trơn tru! Khi gặp lỗi, chụp `pjsip show contacts` + `pjsip set logger on` mình sẽ chỉ điểm ngay.
