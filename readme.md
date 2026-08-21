# Đồ Án Capstone Linux - Hệ Thống Mini Vault

Hệ thống được thiết kế và vận hành theo mô hình phân vùng mạng an toàn sử dụng cơ chế Jump Host (Bastion) và các cơ chế gia cố bảo mật hệ điều hành Linux.

## 1. Hướng đồ án đã chọn
- **Tên dự án:** Mini Vault
- **Công nghệ cốt lõi:**
  - Web Server & Reverse Proxy: Nginx
  - Application Backend: FastAPI (Python) chạy dưới dạng dịch vụ systemd
  - Cơ sở dữ liệu: MySQL chỉ nghe tại `localhost`
  - Hệ điều hành: Ubuntu 22.04.5 LTS Server sạch
  - Nền tảng ảo hóa: Proxmox Virtual Environment (PVE)

## 2. Sơ đồ VM và Phân bổ Mạng
Hạ tầng mạng ảo hóa được chia làm hai phân vùng riêng biệt:
1. **Mạng ngoài (External/WAN) - Bridge `vmbr0`**: Cung cấp IP mạng ngoài cho Jump Host để quản trị viên có thể kết nối SSH từ máy Client cá nhân.
2. **Mạng LAN nội bộ (Private LAN) - Bridge `vmbrcs`**: Cô lập hoàn toàn, dải IP nội bộ `10.10.10.0/24` (Gateway ảo trên Host PVE: `10.10.10.1`).

### Bảng Phân Bổ IP Hệ Thống:
| Tên VM | Hostname | IP WAN / Host Network | IP Private LAN | Cổng Dịch Vụ Mở | Vai Trò |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **VM 200** | `jump-host` | Nhận từ `vmbr0` | `10.10.10.200` | 22 (SSH) | Jump Host (Bastion) trung chuyển |
| **VM 203** | `vm1-svc` | Không có (Isolated) | `10.10.10.11` | 22 (SSH nội bộ), 80/443 (HTTP/HTTPS) | Dịch vụ chính (Web, App, DB) |
| **VM 202** | `vm2-backup` | Không có (Isolated) | `10.10.10.12` | 22 (SSH nội bộ) | Sao lưu lưu trữ chuyên dụng |

### Sơ đồ kết nối:
Mô tả sơ đồ liên kết được biểu diễn qua ảnh sơ đồ tại tệp `images/so_do.png`.

## 3. Bảng phân công thành viên
| STT | Họ và tên | Vai trò phân công thực tế | Mức đóng góp |
| :--- | :--- | :--- | :--- |
| 1 | Cao Lê Gia Phú | Hạ tầng và mạng | 100% |
| 2 | Nguyễn Văn Bảo Phúc | Reverse proxy và bảo mật web | 100% |
| 3 | Nguyễn Lê Nhật Duy | Ứng dụng và systemd | 100% |
| 4 | Lê Nguyễn Nhật Khánh | CSDL và sao lưu | 100% |
| 5 | Lê Hoàng Long | Bộ công cụ tự động hóa và cảnh báo | 100% |

## 4. Cách tái tạo hệ thống
### Bước 1: Thiết lập mạng và Máy ảo trên Proxmox
1. Trên Proxmox VE, tạo một Linux Bridge ảo đặt tên là `vmbrcs` (không gán cổng mạng vật lý), gán địa chỉ IP Gateway: `10.10.10.1/24`.
2. Tạo 3 VM chạy hệ điều hành Ubuntu Server 22.04 LTS:
   - `jump-host` (200): Gán card `net0` vào `vmbr0` (WAN), card `net1` vào `vmbrcs` (IP tĩnh: `10.10.10.200`).
   - `vm1-svc` (203): Gán card `net0` vào `vmbrcs` (IP tĩnh: `10.10.10.11`). Thêm đĩa cứng thứ hai `scsi1` (10 GB).
   - `vm2-backup` (202): Gán card `net0` vào `vmbrcs` (IP tĩnh: `10.10.10.12`). Thêm đĩa cứng thứ hai `scsi1` (20 GB).

### Bước 2: Thiết lập lưu trữ và mount ổ đĩa
Định dạng ổ đĩa thứ hai `ext4` và mount tự động tại `/etc/fstab`:
- Trên `vm1-svc`: Mount ổ cứng `scsi1` vào `/mnt/data`.
- Trên `vm2-backup`: Mount ổ cứng `scsi1` vào `/mnt/backup`.

### Bước 3: Cấu hình SSH ProxyJump
1. Sinh cặp khóa SSH trên máy Client quản trị:
   ```bash
   ssh-keygen -t ed25519 -C "sysadmin@capstone"
   ```
2. Thêm Public Key vào `/home/sysadmin/.ssh/authorized_keys` và `/home/devops/.ssh/authorized_keys` của cả 3 VM.
3. Cấu hình file `~/.ssh/config` trên máy cá nhân để thực hiện ProxyJump qua `jump-host` vào `vm1-svc` và `vm2-backup`.

### Bước 4: Triển khai Nginx Virtual Hosts
1. Cài đặt Nginx trên `vm1-svc`:
   ```bash
   sudo apt install -y nginx
   ```
2. Cấu hình 2 Virtual Host trong `/etc/nginx/sites-available/`:
   - `app.conf`: Proxy tới `http://127.0.0.1:8000` (FastAPI) phục vụ domain `app.lab.local`.
   - `status.conf`: Phục vụ trang trạng thái tĩnh tại thư mục `/var/www/status` cho domain `status.lab.local`.
3. Kích hoạt bằng cách tạo symbolic link vào `/etc/nginx/sites-enabled/` và reload Nginx.

### Bước 5: Thiết lập Dịch vụ Systemd và CSDL
1. Tạo một systemd service file `/etc/systemd/system/myapp.service` chạy ứng dụng FastAPI bằng user không đặc quyền `myapp`.
2. Tạo file biến môi trường `/etc/myapp/myapp.env` (chmod 640) lưu trữ mật khẩu MySQL.
3. Cấu hình MySQL chỉ nghe trên `127.0.0.1` (sửa `bind-address` trong file cấu hình).
4. Tạo CSDL `app_db` và gán quyền tối thiểu (SELECT, INSERT, UPDATE, DELETE) cho tài khoản `app_user` kết nối cục bộ.

### Bước 6: Hardening Hệ Thống
1. Bật tường lửa UFW và chạy tập lệnh `/root/ufw_setup.sh` chỉ cho phép cổng 22, 80, 443.
2. Gia cố SSH trong `/etc/ssh/sshd_config` (vô hiệu hóa root login và password login).
3. Sửa lỗi `cloud-init` ghi đè SSH key bằng cách xóa `/etc/ssh/sshd_config.d/50-cloud-init.conf` và đặt cấu hình `ssh_pwauth: false` tại `/etc/cloud/cloud.cfg.d/99-disable-ssh-pwauth.cfg`.
4. Cài đặt và kích hoạt `fail2ban` bảo vệ SSH.
5. Cài đặt và cấu hình `auditd` theo dõi `/etc/passwd` và `/etc/shadow`.

### Bước 7: Cài đặt Chứng chỉ TLS tự ký (HTTPS)
1. Sinh chứng chỉ tự ký:
   ```bash
   sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
     -keyout /etc/ssl/private/lab-selfsigned.key \
     -out    /etc/ssl/certs/lab-selfsigned.crt \
     -subj "/C=VN/ST=HCMC/O=HCMUS-Lab/CN=app.lab.local"
   ```
2. Cấu hình Nginx chuyển hướng cổng 80 sang cổng 443, và cấu hình lắng nghe SSL ở cổng 443 với tệp chứng chỉ tự ký vừa tạo.
