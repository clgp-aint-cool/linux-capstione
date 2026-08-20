# Vai trò 2 — Reverse Proxy & Bảo mật Web
## Tổng hợp lệnh + nội dung file cấu hình đã dùng (VM1, IP tham khảo: 10.10.10.11)

> Thay `<IP_VM1>`, `10.10.10.11`, tên user bằng giá trị thật của nhóm nếu khác.

---

## 0. Cài gói cần thiết (trên VM1)

```bash
sudo apt update
sudo apt install -y nginx ufw fail2ban auditd audispd-plugins lynis openssl curl sshpass
```

---

## 1. Tạo user quản trị (đứng ở root@VM1)

```bash
# Tạo user quản trị SSH
useradd -m -s /bin/bash sysadmin
passwd sysadmin
usermod -aG sudo sysadmin

useradd -m -s /bin/bash devops
passwd devops
usermod -aG sudo devops

# Xác nhận
getent passwd sysadmin devops
```

---

## 2. Tạo SSH keypair (trên máy CLIENT — Jumphost, KHÔNG phải trên VM1)

```bash
ssh-keygen -t ed25519 -C "sysadmin@capstone-svc01"
cat ~/.ssh/id_ed25519.pub
```

---

## 3. Gắn public key vào từng user (đứng ở root@VM1, không cần ssh-copy-id)

```bash
# Cho sysadmin — thay dòng ssh-ed25519 bằng key thật lấy từ Bước 2
mkdir -p /home/sysadmin/.ssh
cat >> /home/sysadmin/.ssh/authorized_keys << 'EOF'
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGx7k3... sysadmin@capstone-svc01
EOF
chmod 700 /home/sysadmin/.ssh
chmod 600 /home/sysadmin/.ssh/authorized_keys
chown -R sysadmin:sysadmin /home/sysadmin/.ssh

# Cho devops (dùng chung key tạm thời để test, thay key thật của thành viên sau)
mkdir -p /home/devops/.ssh
cat >> /home/devops/.ssh/authorized_keys << 'EOF'
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIGx7k3... sysadmin@capstone-svc01
EOF
chmod 700 /home/devops/.ssh
chmod 600 /home/devops/.ssh/authorized_keys
chown -R devops:devops /home/devops/.ssh

# Xác nhận quyền đúng chuẩn
ls -la /home/sysadmin/.ssh/ /home/devops/.ssh/
```

### Test key hoạt động (chạy từ máy CLIENT, terminal MỚI, giữ nguyên session root cũ)

```bash
ssh -v sysadmin@10.10.10.11
# tìm dòng: debug1: Authentication succeeded (publickey).

sudo whoami
# kỳ vọng: root

ssh -v devops@10.10.10.11
```

---

## 4. Nginx — 2 Virtual Host

### 4.1 Trang trạng thái tĩnh (vhost 2)

```bash
mkdir -p /var/www/status
cat > /var/www/status/index.html << 'EOF'
<!DOCTYPE html>
<html lang="vi">
<head>
<meta charset="UTF-8">
<title>Trạng thái hệ thống - Capstone</title>
<style>
  body { font-family: sans-serif; background:#0d1117; color:#c9d1d9; padding:2rem; }
  h1 { color:#58a6ff; }
  .ok { color:#3fb950; font-weight:bold; }
</style>
</head>
<body>
  <h1>Capstone System Status</h1>
  <p>Vhost 2 - trang trạng thái tĩnh phục vụ bởi Nginx.</p>
  <p>Trạng thái web server: <span class="ok">ONLINE</span></p>
  <p>Xem chi tiết dịch vụ qua bộ công cụ health-check (scripts/healthcheck.sh).</p>
</body>
</html>
EOF
```

### 4.2 Vhost 1 — reverse proxy tới app FastAPI

```bash
cat > /etc/nginx/sites-available/app.conf << 'EOF'
# /etc/nginx/sites-available/app.conf
# Vhost 1: proxy tới ứng dụng FastAPI (uvicorn, chỉ nghe 127.0.0.1:8000)

server {
    listen 80;
    server_name app.lab.local;

    access_log /var/log/nginx/app.access.log;
    error_log  /var/log/nginx/app.error.log;

    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
```

### 4.3 Vhost 2 — trang trạng thái tĩnh

```bash
cat > /etc/nginx/sites-available/status.conf << 'EOF'
# /etc/nginx/sites-available/status.conf
# Vhost 2: trang trạng thái tĩnh - tổng cộng đủ >=2 virtual host trên cùng Nginx

server {
    listen 80;
    server_name status.lab.local;

    root /var/www/status;
    index index.html;

    access_log /var/log/nginx/status.access.log;
    error_log  /var/log/nginx/status.error.log;

    location / {
        try_files $uri $uri/ =404;
    }
}
EOF
```

### 4.4 Kích hoạt cả 2 vhost

```bash
ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/
ln -s /etc/nginx/sites-available/status.conf /etc/nginx/sites-enabled/
rm -f /etc/nginx/sites-enabled/default

nginx -t
systemctl reload nginx
```

### 4.5 Test 2 vhost

```bash
curl -H "Host: app.lab.local" http://127.0.0.1/
curl -H "Host: status.lab.local" http://127.0.0.1/

# Test từ máy khác (VM2/host) — đúng yêu cầu demo §4.3
curl -H "Host: app.lab.local" http://10.10.10.11/
curl -H "Host: status.lab.local" http://10.10.10.11/
```

---

## 5. UFW Firewall

```bash
cat > /root/ufw_setup.sh << 'EOF'
#!/usr/bin/env bash
# ufw_setup.sh - Cấu hình tường lửa UFW: mặc định chặn vào, chỉ mở cổng cần thiết.
set -euo pipefail
trap 'echo "[LỖI] dòng $LINENO thất bại" >&2' ERR

SSH_PORT="${1:-22}"   # cho phép đổi cổng SSH khi gọi: ./ufw_setup.sh 2222

echo "[*] Đặt chính sách mặc định: deny incoming, allow outgoing"
ufw default deny incoming
ufw default allow outgoing

echo "[*] Mở cổng SSH (${SSH_PORT}/tcp) - quản trị từ xa"
ufw allow "${SSH_PORT}/tcp" comment 'SSH quan tri'

echo "[*] Mở cổng 80/tcp (HTTP) - Nginx reverse proxy + vhost trạng thái"
ufw allow 80/tcp comment 'HTTP Nginx'

echo "[*] Mở cổng 443/tcp (HTTPS) - chỉ cần nếu làm điểm thưởng TLS"
ufw allow 443/tcp comment 'HTTPS Nginx TLS'

# LƯU Ý: KHÔNG mở 5432 (PostgreSQL) và KHÔNG mở 8000 (uvicorn) ra ngoài -
# hai dịch vụ này chỉ nghe 127.0.0.1, không cần và không nên lộ ra tường lửa.

echo "[*] Bật UFW"
ufw --force enable
ufw status verbose
EOF
chmod +x /root/ufw_setup.sh
bash /root/ufw_setup.sh 22
```

---

## 6. SSH Hardening

### 6.1 Sửa `/etc/ssh/sshd_config`

Thêm/sửa các dòng (dùng `nano /etc/ssh/sshd_config` rồi sửa tay, hoặc thêm khối này vào cuối file):

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
AllowUsers sysadmin devops
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
```

### 6.2 Test cú pháp & giá trị hiệu lực thật

```bash
sshd -t
sshd -T | grep -i passwordauthentication
```

### 6.3 Bug gặp phải: cloud-init ghi đè `PasswordAuthentication` thành `yes`

```bash
# Phát hiện thủ phạm
ls -la /etc/ssh/sshd_config.d/
grep -rn "PasswordAuthentication" /etc/ssh/sshd_config.d/ /etc/ssh/sshd_config

# Xử lý — xóa hẳn file override của cloud-init
rm /etc/ssh/sshd_config.d/50-cloud-init.conf

# Chặn cloud-init tự sinh lại file này ở lần boot sau
mkdir -p /etc/cloud/cloud.cfg.d
echo "ssh_pwauth: false" > /etc/cloud/cloud.cfg.d/99-disable-ssh-pwauth.cfg

# Xác nhận lại
sshd -t
sshd -T | grep -i passwordauthentication
# kỳ vọng: passwordauthentication no
```

### 6.4 Restart & test đầy đủ (bắt buộc trước khi đóng session root cũ)

```bash
systemctl restart ssh
```

Từ terminal MỚI (giữ session root cũ):
```bash
ssh -v sysadmin@10.10.10.11
# kỳ vọng dòng: "Authentications that can continue: publickey" (KHÔNG còn password)
# và: "Authentication succeeded (publickey)."

sudo whoami   # kỳ vọng: root

ssh -v devops@10.10.10.11

# Xác nhận root bị chặn hoàn toàn
ssh root@10.10.10.11
# kỳ vọng: Permission denied (publickey).

```

---

## 7. fail2ban

```bash
cat > /etc/fail2ban/jail.local << 'EOF'
# /etc/fail2ban/jail.local
# Cấu hình fail2ban bảo vệ SSH khỏi dò mật khẩu.

[DEFAULT]
bantime  = 600
findtime = 300
maxretry = 3
backend  = systemd

[sshd]
enabled  = true
port     = ssh
logpath  = %(sshd_log)s
maxretry = 3
bantime  = 600

# Nếu app có endpoint đăng nhập, thêm jail riêng để ban theo log lỗi 401/403 của Nginx:
# [nginx-app-auth]
# enabled  = true
# port     = http,https
# filter   = nginx-app-auth
# logpath  = /var/log/nginx/app.error.log
# maxretry = 5
# bantime  = 600
EOF

systemctl enable --now fail2ban
systemctl status fail2ban --no-pager
```

### Script mô phỏng brute-force (tạo & chạy trên VM2, KHÔNG chạy trên VM1)

```bash
# Chạy trên VM2
cat > ~/simulate_ssh_bruteforce.sh << 'EOF'
#!/usr/bin/env bash
# simulate_ssh_bruteforce.sh - Mô phỏng dò mật khẩu SSH TỪ MÁY KHÁC (VM2 hoặc host)
# để kích hoạt fail2ban ban trực tiếp lúc demo.
set -euo pipefail

TARGET_HOST="${1:?Cần truyền IP của VM1, vd: ./simulate_ssh_bruteforce.sh 192.168.56.10}"
TARGET_PORT="${2:-22}"

echo "[*] Thử SSH sai mật khẩu 4 lần vào ${TARGET_HOST}:${TARGET_PORT} để kích hoạt fail2ban..."
for i in 1 2 3 4; do
  echo "  -> lần $i"
  sshpass -p "matkhau-sai-co-y-$i" ssh -p "${TARGET_PORT}" \
    -o StrictHostKeyChecking=no -o ConnectTimeout=3 \
    fake_user@"${TARGET_HOST}" exit || true
  sleep 1
done

echo "[*] Trên VM1, kiểm tra ban bằng:"
echo "    sudo fail2ban-client status sshd"
EOF
chmod +x ~/simulate_ssh_bruteforce.sh

sudo apt install -y sshpass
bash ~/simulate_ssh_bruteforce.sh 10.10.10.11 22
```

### Kiểm tra kết quả (trên VM1)

```bash
fail2ban-client status sshd
# kỳ vọng thấy IP VM2 trong "Banned IP list"

# Unban để test lại
fail2ban-client set sshd unbanip <IP_VM2>
```

---

## 8. auditd

```bash
cat > /etc/audit/rules.d/capstone.rules << 'EOF'
# /etc/audit/rules.d/capstone.rules
# Theo dõi truy cập/đổi 2 tệp nhạy cảm bắt buộc theo đề (§3.2).

-w /etc/passwd -p wa -k capstone_passwd_watch
-w /etc/shadow -p wa -k capstone_shadow_watch

# Xem log kiểm toán:
#   ausearch -k capstone_passwd_watch
#   ausearch -k capstone_shadow_watch
EOF

augenrules --load
systemctl restart auditd

# Xác nhận luật đã nạp
auditctl -l
```

### Test bắt sự kiện thật (dùng attribute-change, KHÔNG dùng read)

```bash
chmod 644 /etc/passwd && chmod 644 /etc/passwd
ausearch -k capstone_passwd_watch -ts recent

# Hoặc test bằng useradd (ghi thật)
useradd -M -N test_audit_demo
ausearch -k capstone_passwd_watch -ts recent
userdel test_audit_demo
```

---

## 9. Lynis

```bash
lynis audit system
# ghi lại điểm Hardening index + đọc các suggestion

# Ví dụ 3 mục khắc phục thường gặp:
sed -i 's/^PASS_MAX_DAYS.*/PASS_MAX_DAYS   90/' /etc/login.defs
sed -i 's/^PASS_MIN_DAYS.*/PASS_MIN_DAYS   7/'  /etc/login.defs

echo "Authorized access only. All activity is monitored." | tee /etc/issue.net
echo 'Banner /etc/issue.net' | tee -a /etc/ssh/sshd_config
systemctl restart ssh

echo "net.ipv6.conf.all.disable_ipv6 = 1"     | tee -a /etc/sysctl.conf
echo "net.ipv6.conf.default.disable_ipv6 = 1" | tee -a /etc/sysctl.conf
sysctl -p

# Chạy lại để xác nhận điểm tăng
lynis audit system
```

---

## 10. Lệnh kiểm tra tổng hợp trước demo

```bash
nginx -t
ufw status verbose
sshd -T | grep -iE "permitrootlogin|passwordauthentication|allowusers"
systemctl status nginx ufw fail2ban auditd --no-pager
fail2ban-client status sshd
auditctl -l
curl -H "Host: app.lab.local" http://10.10.10.11/
curl -H "Host: status.lab.local" http://10.10.10.11/
```

---

## Ghi chú quan trọng cho báo cáo (Mục 7 — Nhìn lại)

- **Bug cloud-init:** VM tạo từ cloud image trên Proxmox tự sinh `/etc/ssh/sshd_config.d/50-cloud-init.conf` chứa `PasswordAuthentication yes`, ghi đè cấu hình `no` trong `sshd_config` chính do cơ chế "giá trị đọc trước thắng" của `Include`. Phát hiện bằng `sshd -T` (giá trị hiệu lực thật) thay vì chỉ đọc file cấu hình tĩnh. Khắc phục bằng cách xóa file override + thêm `ssh_pwauth: false` vào `/etc/cloud/cloud.cfg.d/` để chặn tái phát sinh.
