# Hướng dẫn cài AlmaLinux 9.6 Minimal trên Proxmox và chuẩn bị Template

## 1. Chuẩn bị
- Tải ISO AlmaLinux 9.6 Minimal từ: [https://almalinux.org](https://almalinux.org)
- Upload ISO vào Proxmox: **Datacenter → Storage → ISO Images → Upload**

---

## 2. Tạo VM trên Proxmox
1. Nhấn **Create VM**
2. **General**: Name = `almalinux9`
3. **OS**: ISO image = `AlmaLinux-9.6-x86_64-minimal.iso`, Guest OS = Linux
4. **System**: BIOS = `SeaBIOS` hoặc `OVMF`, Machine = `q35`
5. **Hard Disk**: Bus/Device = VirtIO, Disk size = 20GB
6. **CPU**: 2–4 vCPU (Sockets=1, Cores=2–4)
7. **Memory**: 2048–4096 MB
8. **Network**: VirtIO, Bridge = vmbr0
9. **Confirm → Finish**

---

## 3. Cài đặt AlmaLinux trong VM
1. Boot từ ISO → chọn **Install AlmaLinux 9.6**
2. Trong **Installation Summary**:
   - Keyboard = English (US)
   - Timezone = Asia/Ho_Chi_Minh
   - Installation Destination = Automatic
   - Software Selection = **Minimal Install**
   - Network & Hostname = Bật card mạng, đặt hostname
   - Root Password = đặt mật khẩu root
   - (Optional) tạo user admin
3. Nhấn **Begin Installation**
4. Cài xong → **Reboot**

---

## 4. Đăng nhập AlmaLinux
Đăng nhập bằng root tại console hoặc PuTTY:
```
login: root
password: <mật khẩu bạn đặt>
```

---

## 5. Cấu hình hệ thống cơ bản

### Bật SSH server
```bash
systemctl enable sshd --now
```

### Update hệ thống
```bash
dnf update -y
```

### Cài cloud-init
```bash
dnf install -y cloud-init
systemctl enable cloud-init
```

### Cài cmd.log để log lại lệnh user
```bash
echo 'export PROMPT_COMMAND="history -a >(tee -a /var/log/cmd.log)"' > /etc/profile.d/cmdlog.sh
```

---

## 6. Chuẩn bị để clone VM (reset ID, keys)

### Reset machine-id
```bash
truncate -s 0 /etc/machine-id
```

(Tuỳ chọn, nếu muốn symlink như CentOS cũ)
```bash
mkdir -p /var/lib/dbus
ln -s /etc/machine-id /var/lib/dbus/machine-id
```

### Làm sạch cloud-init
```bash
cloud-init clean -s -l
```

### Xoá SSH host keys
```bash
rm -f /etc/ssh/ssh_host_*
```

### Tắt máy
```bash
shutdown -h now
```

---

## 7. Convert VM thành Template trên Proxmox
- Trong Proxmox Web → Chuột phải VM → **Convert to Template**
- Khi cần tạo VM mới → **Clone** từ template này

---

## ✅ Kết quả
- VM có **cloud-init** để tự động cấu hình
- Có **cmd.log** để log lại lệnh user
- **machine-id** và **SSH key** sẽ tự sinh mới khi clone
- Không lo bị trùng ID hoặc key khi tạo nhiều VM
