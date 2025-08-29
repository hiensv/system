# HƯỚNG DẪN CÀI DEBIAN TRÊN VM PROXMOX

## 1. Chuẩn bị
- **ISO Debian**: tải từ [https://www.debian.org/distrib/](https://www.debian.org/distrib/) (chọn bản `netinst` ~ 600MB).
- **Proxmox VE**: đã cài sẵn trên server.
- **Tạo VM** trong Proxmox:
  - CPU: 2 cores
  - RAM: 2–4GB
  - Disk: ≥ 20GB (SCSI, virtio-scsi-single, iothread=on)
  - NIC: virtio, bridge `vmbr0` hoặc VLAN của bạn
  - ISO: chọn file Debian netinst đã upload

---

## 2. Cài đặt Debian

### Bước 1: Boot từ ISO
- Start VM → Console → chọn **Graphical install**.

### Bước 2: Cấu hình cơ bản
- **Language**: English (hoặc Vietnamese)
- **Country**: Vietnam
- **Keyboard**: American English

### Bước 3: Network
- Hostname: `debian-voip`
- Domain: để trống hoặc `local`

### Bước 4: Root & User
- **Root password**: đặt mật khẩu mạnh (`Root@2024!`)
- **Create a user**: ví dụ user `admin`
- Password cho user: `Admin@2024!`

### Bước 5: Timezone
- Chọn Asia → Ho Chi Minh

### Bước 6: Partition Disk
- Guided – use entire disk
- All files in one partition
- Finish → Yes

### Bước 7: Package selection
- Bỏ chọn **Desktop environment**
- Chỉ chọn:
  - `SSH server`
  - `standard system utilities`

### Bước 8: GRUB bootloader
- Chọn **Yes** để cài GRUB
- Chọn ổ `/dev/sda` (hoặc disk chính)

---

## 3. Sau khi cài xong
- VM reboot → bạn sẽ thấy màn hình login Debian.
- Đăng nhập bằng:
  - user: `root` hoặc `admin`
  - password: như đã đặt

---

## 4. Cấu hình sau cài đặt
```bash
# Cập nhật hệ thống
apt update && apt upgrade -y

# Cài SSH (nếu chưa có)
apt install openssh-server -y

# Kiểm tra IP
ip a

# Đặt timezone về Việt Nam (nếu sai)
timedatectl set-timezone Asia/Ho_Chi_Minh
```

---

## 5. Kết nối từ ngoài
- Dùng **SSH client** (Putty, MobaXterm, hoặc `ssh` trên Linux/Mac):
  ```bash
  ssh admin@<IP_VM>
  ```

---

✅ Giờ Debian đã sẵn sàng để bạn cài **Asterisk 22** hoặc các dịch vụ khác.
