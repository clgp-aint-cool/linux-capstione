```markdown
# Vai trò 5 — Vận hành & Tự động hóa (Thành phần 3)

## Tổng hợp lệnh, mã nguồn bộ công cụ `ops-tool` & kịch bản kiểm thử (VM1: 10.10.10.11 -> VM2: 10.10.10.12)

Thư mục làm việc chuẩn: `/home/sysadmin/ops-tool/`. Bộ công cụ chịu trách nhiệm điều phối toàn bộ vòng đời vận hành: Triển khai an toàn có Auto-Rollback, Sao lưu 3-2-1 sang VM2, Khôi phục thảm họa, Giám sát cảnh báo Telegram, và Xoay vòng log.
```
---

## 0. Cài đặt các gói công cụ bổ trợ (trên VM1)

```bash
sudo apt update
sudo apt install -y shellcheck rsync curl tar gzip cron jq
```

---

## 1. Cấu hình xác thực SSH Key sang VM2 (Sao lưu 3-2-1)
Thiết lập SSH Key không cần mật khẩu từ tài khoản `sysadmin` trên VM1 sang tài khoản `vmbackup` trên VM2 để phục vụ tiến trình `rsync` tự động:

```bash
# Đứng ở user sysadmin@VM1
ssh-keygen -t ed25519 -N "" -f ~/.ssh/id_ed25519
ssh-copy-id vmbackup@10.10.10.12

# Kiểm tra kết nối không cần mật khẩu sang VM2
ssh -o StrictHostKeyChecking=no vmbackup@10.10.10.12 "echo '[OK] Ket noi SSH VM1 -> VM2 thanh cong!'"
```
---

## 2. Khởi tạo cấu trúc thư mục & Tệp cấu hình môi trường

### 2.1 Tạo khung thư mục bộ công cụ
```bash
mkdir -p /home/sysadmin/ops-tool/{scripts,lib,logs}
sudo mkdir -p /data/backups
sudo chown -R sysadmin:sysadmin /data/backups /home/sysadmin/ops-tool
```

### 2.2 Tệp cấu hình biến môi trường (`config.env`)
```bash
# /home/sysadmin/ops-tool/config.env
# Telegram Bot Configuration
export TELEGRAM_BOT_TOKEN="<TELEGRAM_BOT_TOKEN>"
export TELEGRAM_CHAT_ID="<TELEGRAM_CHAT_ID>"

# System Thresholds (%)
export THRESHOLD_CPU=80
export THRESHOLD_DISK=80
export THRESHOLD_RAM=85

# Backup Configuration
export BACKUP_LOCAL_DIR="/data/backups"
export BACKUP_REMOTE_HOST="10.10.10.12"
export BACKUP_REMOTE_USER="vmbackup"
export BACKUP_REMOTE_DIR="/home/vmbackup/backups"
export BACKUP_RETENTION_DAYS=7

# Change time to VN
export TZ="Asia/Ho_Chi_Minh"
```
```
chmod 600 /home/sysadmin/ops-tool/config.env
```

### 2.3 Thư viện gửi cảnh báo Telegram (`lib/alert.sh`)
```bash
#!/usr/bin/env bash
set -euo pipefail

# Hàm gửi cảnh báo Telegram
send_telegram_alert() {
    local message="$1"

    if [[ -z "${TELEGRAM_BOT_TOKEN:-}" || -z "${TELEGRAM_CHAT_ID:-}" ]]; then
        echo "[ERROR] Telegram token hoặc Chat ID chưa được cấu hình." >&2
        return 1
    fi

    local url="https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage"

    if ! curl -fsS -X POST "$url" \
        --data-urlencode "chat_id=${TELEGRAM_CHAT_ID}" \
        --data-urlencode "text=${message}" \
        > /dev/null; then
        echo "[ERROR] Không thể gửi Telegram alert." >&2
        return 1
    fi
}
```

---

## 3. Bộ script vận hành chuyên biệt (`scripts/`)
### 3.1 Script Giám sát hệ thống & Endpoint (`scripts/healthcheck.sh`)

Tự động quét CPU, RAM, dung lượng đĩa / và /data, 3 dịch vụ, 4 cổng và kiểm tra chuỗi dịch vụ Web $\rightarrow$ App $\rightarrow$ DB qua endpoint `https://127.0.0.1/health`

**Cách chạy CLI:**
```
./home/sysadmin/ops-tool/scripts/healthcheck.sh
```
**Scipt:**
```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

# shellcheck source=/dev/null
source "${BASE_DIR}/config.env"
# shellcheck source=/dev/null
source "${BASE_DIR}/lib/alert.sh"

LOG_FILE="${BASE_DIR}/logs/healthcheck.log"
mkdir -p "$(dirname "$LOG_FILE")"

cleanup_on_err() {
    local exit_code=$?
    local line_no=$1
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [FATAL] Script gặp lỗi tại dòng ${line_no} với mã thoát ${exit_code}" | tee -a "$LOG_FILE"
}
trap 'cleanup_on_err ${LINENO}' ERR

log_msg() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] ${message}" | tee -a "$LOG_FILE"
}

has_issues=0
alert_body=""

# 1. KIỂM TRA DUNG LƯỢNG Ổ ĐĨA
check_disk() {
    local mount_point="$1"
    if mountpoint -q "$mount_point" || [[ "$mount_point" == "/" ]]; then
        local usage
        usage=$(df -P "$mount_point" | awk 'NR==2 {print $5}' | tr -d '%')
        if (( usage >= THRESHOLD_DISK )); then
            local issue="⚠️ Ổ đĩa ${mount_point} vượt ngưỡng: ${usage}% (Ngưỡng: ${THRESHOLD_DISK}%)"
            log_msg "WARN" "$issue"
            alert_body+="${issue}\n"
            has_issues=1
        else
            log_msg "INFO" "Ổ đĩa ${mount_point} bình thường: ${usage}%"
        fi
    fi
}

# 2. KIỂM TRA RAM
check_ram() {
    local ram_usage
    ram_usage=$(free | awk '/Mem:/ {printf "%.0f", ($3/$2)*100}')
    if (( ram_usage >= THRESHOLD_RAM )); then
        local issue="⚠️ RAM vượt ngưỡng: ${ram_usage}% (Ngưỡng: ${THRESHOLD_RAM}%)"
        log_msg "WARN" "$issue"
        alert_body+="${issue}\n"
        has_issues=1
    else
        log_msg "INFO" "RAM bình thường: ${ram_usage}%"
    fi
}

# 3. Kiểm tra % CPU
check_cpu() {
    local cpu_idle
    cpu_idle=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print int($1)}')
    local cpu_usage=$(( 100 - cpu_idle ))
    if (( cpu_usage >= THRESHOLD_CPU )); then
        local issue="⚠️ CPU vượt ngưỡng: ${cpu_usage}% (Ngưỡng: ${THRESHOLD_CPU}%)"
        log_msg "WARN" "$issue"
        alert_body+="${issue}\n"
        has_issues=1
    else
        log_msg "INFO" "CPU bình thường: ${cpu_usage}%"
    fi
}

# 4. KIỂM TRA DỊCH VỤ SYSTEMD
check_service() {
    local svc="$1"
    if systemctl is-active --quiet "$svc"; then
        log_msg "INFO" "Dịch vụ ${svc}: RUNNING"
    else
        local issue="🚨 Dịch vụ ${svc} đang BỊ DỪNG!"
        log_msg "ERROR" "$issue"
        alert_body+="${issue}\n"
        has_issues=1
    fi
}

# 5. KIỂM TRA CỔNG MẠNG LẮNG NGHE
check_port() {
    local port="$1"
    local desc="$2"
    if ss -tulpn | grep -q ":${port} "; then
        log_msg "INFO" "Cổng ${port} (${desc}): Đang lắng nghe"
    else
        local issue="🚨 Cổng ${port} (${desc}) KHÔNG phản hồi!"
        log_msg "ERROR" "$issue"
        alert_body+="${issue}\n"
        has_issues=1
    fi
}

# 6. KIỂM TRA ENDPOINT ĐẦU-CUỐI QUA HTTPS
check_http_endpoint() {
    local endpoint_url="https://127.0.0.1/health"
    local host_header="app.lab.local"
    local response

    if response=$(curl -k -fsS -H "Host: ${host_header}" "${endpoint_url}" 2>/dev/null); then
        log_msg "INFO" "Endpoint /health phản hồi tốt: ${response}"
    else
        local issue="🚨 Endpoint HTTPS /health không phản hồi hoặc trả về lỗi!"
        log_msg "ERROR" "$issue"
        alert_body+="${issue}\n"
        has_issues=1
    fi
}

# --- THỰC THI KIỂM TRA ---
log_msg "INFO" "=== BẮT ĐẦU HEALTH CHECK ==="

check_disk "/"
check_disk "/data"
check_ram
check_cpu

# Kiểm tra đầy đủ 3 dịch vụ
check_service "nginx"
check_service "mysql"
check_service "myapp"

# Kiểm tra các cổng theo thiết kế
check_port "80" "HTTP Nginx"
check_port "443" "HTTPS Nginx"
check_port "8000" "Gunicorn App"
check_port "3306" "MySQL"

# Kiểm tra chuỗi dịch vụ Web -> App -> DB
check_http_endpoint

if (( has_issues == 1 )); then
    full_alert="🚨 [VM1 - HEALTH CHECK ALERT] Phát hiện sự cố:\n\n$(echo -e "$alert_body")"
    send_telegram_alert "$full_alert"
    log_msg "WARN" "Đã gửi cảnh báo sự cố về Telegram."
else
    log_msg "INFO" "Tất cả các thành phần hệ thống hoạt động bình thường."
fi

log_msg "INFO" "=== HOÀN TẤT HEALTH CHECK ==="
```

---

### 3.2 Script Sao lưu toàn diện đa tầng & Rsync sang VM2 (`scripts/backup.sh`)

Tự động kích hoạt `db_backup.sh`, gom mã nguồn `/opt/myapp`, cấu hình Nginx, chứng chỉ TLS và file secrets, nén `.tar.gz`, đẩy sang VM2 và xóa bản lưu quá 7 ngày

**Cách chạy CLI:**
```
./home/sysadmin/ops-tool/scripts/backup.sh
```
**Scipt:**
```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"
# shellcheck source=/dev/null
source "${BASE_DIR}/config.env"
# shellcheck source=/dev/null
source "${BASE_DIR}/lib/alert.sh"

LOG_FILE="${BASE_DIR}/logs/backup.log"
mkdir -p "$(dirname "$LOG_FILE")"

cleanup_on_err() {
    local exit_code=$?
    local line_no=$1
    local err_msg="🚨 [VM1] Backup THẤT BẠI tại dòng ${line_no} (Exit: ${exit_code})"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [FATAL] ${err_msg}" | tee -a "$LOG_FILE"
    send_telegram_alert "$err_msg"
}
trap 'cleanup_on_err ${LINENO}' ERR

log_msg() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] ${message}" | tee -a "$LOG_FILE"
}

log_msg "INFO" "=== BẮT ĐẦU QUY TRÌNH SAO LƯU TOÀN DIỆN ==="

TIMESTAMP="$(date '+%Y%m%d_%H%M%S')"
BACKUP_NAME="backup_${TIMESTAMP}"
TEMP_BACKUP_DIR="/tmp/${BACKUP_NAME}"
ARCHIVE_FILE="${BACKUP_LOCAL_DIR}/${BACKUP_NAME}.tar.gz"

sudo rm -rf "$TEMP_BACKUP_DIR"
mkdir -p "$TEMP_BACKUP_DIR"
sudo mkdir -p "$BACKUP_LOCAL_DIR"
sudo chown -R sysadmin:sysadmin "$BACKUP_LOCAL_DIR"

# 1. GỌI SCRIPT DUMP CSDL (TẦNG DB)
mkdir -p "${TEMP_BACKUP_DIR}/database"
DB_SCRIPT="/home/sysadmin/scripts/scripts_backup_restore_DB/db_backup.sh"

if [[ -f "$DB_SCRIPT" ]]; then
    log_msg "INFO" "Kích hoạt script sao lưu CSDL..."
    sudo bash "$DB_SCRIPT"

    if [[ -f "/etc/db_backup.env" ]]; then
        # shellcheck source=/dev/null
        source "/etc/db_backup.env"
        LATEST_SQL=$(find "${BACKUP_DIR}" -maxdepth 1 -name "${DB_NAME}_*.sql.gz" -printf "%T@ %p\n" | sort -nr | head -n 1 | cut -d' ' -f2-)
        if [[ -n "$LATEST_SQL" && -f "$LATEST_SQL" ]]; then
            cp "$LATEST_SQL" "${TEMP_BACKUP_DIR}/database/"
            log_msg "INFO" "Đã đưa bản dump DB ($(basename "$LATEST_SQL")) vào gói sao lưu."
        fi
    fi
fi

# 2. SAO LƯU MÃ NGUỒN APP, SECRETS & SYSTEMD (TẦNG APP)
mkdir -p "${TEMP_BACKUP_DIR}/app"
if [[ -d "/opt/myapp" ]]; then
    sudo rsync -a --exclude 'venv' /opt/myapp/ "${TEMP_BACKUP_DIR}/app/code/"
fi

if [[ -f "/etc/myapp/myapp.env" ]]; then
    mkdir -p "${TEMP_BACKUP_DIR}/app/secrets"
    sudo cp /etc/myapp/myapp.env "${TEMP_BACKUP_DIR}/app/secrets/"
fi

mkdir -p "${TEMP_BACKUP_DIR}/systemd"
if [[ -f "/etc/systemd/system/myapp.service" ]]; then
    sudo cp /etc/systemd/system/myapp.service "${TEMP_BACKUP_DIR}/systemd/"
fi

# 3. SAO LƯU CẤU HÌNH WEB & TLS (TẦNG NGINX)
mkdir -p "${TEMP_BACKUP_DIR}/web"
sudo cp -r /etc/nginx "${TEMP_BACKUP_DIR}/web/nginx_conf"
if [[ -d "/var/www" ]]; then
    sudo cp -r /var/www "${TEMP_BACKUP_DIR}/web/www_data"
fi

mkdir -p "${TEMP_BACKUP_DIR}/web/ssl"
[[ -f "/etc/ssl/certs/lab-selfsigned.crt" ]] && sudo cp /etc/ssl/certs/lab-selfsigned.crt "${TEMP_BACKUP_DIR}/web/ssl/"
[[ -f "/etc/ssl/private/lab-selfsigned.key" ]] && sudo cp /etc/ssl/private/lab-selfsigned.key "${TEMP_BACKUP_DIR}/web/ssl/"

# 4. ĐÓNG GÓI VÀ NÉN
log_msg "INFO" "Đóng gói toàn bộ hệ thống vào file: ${ARCHIVE_FILE}..."
sudo tar -czf "$ARCHIVE_FILE" -C "/tmp" "$BACKUP_NAME"
sudo chown sysadmin:sysadmin "$ARCHIVE_FILE"
sudo rm -rf "$TEMP_BACKUP_DIR"
log_msg "INFO" "Tạo bản sao lưu thành công: $(basename "$ARCHIVE_FILE") ($(du -h "$ARCHIVE_FILE" | cut -f1))"

# 5. ĐỒNG BỘ SANG VM2 (10.10.10.12)
log_msg "INFO" "Đồng bộ bản sao lưu sang VM2 (${BACKUP_REMOTE_HOST})..."
rsync -avz -e "ssh -o StrictHostKeyChecking=no" "$ARCHIVE_FILE" "${BACKUP_REMOTE_USER}@${BACKUP_REMOTE_HOST}:${BACKUP_REMOTE_DIR}/"
log_msg "INFO" "Đồng bộ sang VM2 thành công."

# 6. RETENTION (DỌN DẸP BẢN LƯU CŨ HƠN 7 NGÀY)
log_msg "INFO" "Dọn dẹp bản sao lưu cũ hơn ${BACKUP_RETENTION_DAYS} ngày..."
find "$BACKUP_LOCAL_DIR" -name "backup_*.tar.gz" -type f -mtime +"${BACKUP_RETENTION_DAYS}" -exec rm -f {} \;

log_msg "INFO" "=== SAO LƯU HOÀN TẤT THÀNH CÔNG ==="
send_telegram_alert "✅ *[VM1]* Đã sao lưu toàn diện \`${BACKUP_NAME}.tar.gz\` (MySQL + App + Web + TLS) và đồng bộ an toàn sang VM2!"
```

---

### 3.3 Script Khôi phục thảm họa toàn diện (`scripts/restore.sh`)
Giải nén bản sao lưu, phục hồi cấu hình Nginx/TLS, mã nguồn `/opt/myapp`, nạp biến môi trường và gọi `db_restore.sh` nạp lại CSDL MySQL

**Cách chạy CLI:**
```
./home/sysadmin/ops-tool/scripts/restore.sh --help
Sử dụng: restore.sh [TÙY CHỌN]

Công cụ khôi phục toàn diện hệ thống từ bản sao lưu .tar.gz.

Tùy chọn:
  -f <đường_dẫn_file>   Đường dẫn tới file backup_*.tar.gz cần khôi phục
  -l                    Khôi phục từ bản sao lưu MỚI NHẤT trong /data/backups
  -h, --help            Hiển thị hướng dẫn này

Ví dụ:
  restore.sh -l
```
**Scipt:**
```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

# shellcheck source=/dev/null
source "${BASE_DIR}/config.env"
# shellcheck source=/dev/null
source "${BASE_DIR}/lib/alert.sh"

LOG_FILE="${BASE_DIR}/logs/restore.log"
mkdir -p "$(dirname "$LOG_FILE")"

cleanup_on_err() {
    local exit_code=$?
    local line_no=$1
    local err_msg="🚨 [VM1] Khôi phục THẤT BẠI tại dòng ${line_no} (Exit: ${exit_code})"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [FATAL] ${err_msg}" | tee -a "$LOG_FILE"
    send_telegram_alert "$err_msg"
}
trap 'cleanup_on_err ${LINENO}' ERR

log_msg() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] ${message}" | tee -a "$LOG_FILE"
}

show_help() {
    cat << HELP_EOF
Sử dụng: $(basename "$0") [TÙY CHỌN]

Công cụ khôi phục toàn diện hệ thống từ bản sao lưu .tar.gz.

Tùy chọn:
  -f <đường_dẫn_file>   Đường dẫn tới file backup_*.tar.gz cần khôi phục
  -l                    Khôi phục từ bản sao lưu MỚI NHẤT trong /data/backups
  -h, --help            Hiển thị hướng dẫn này

Ví dụ:
  $(basename "$0") -l
HELP_EOF
    exit 0
}

TARGET_FILE=""
USE_LATEST=0

for arg in "$@"; do
    if [[ "$arg" == "--help" ]]; then
        show_help
    fi
done

while getopts ":f:lh" opt; do
    case "$opt" in
        f) TARGET_FILE="$OPTARG" ;;
        l) USE_LATEST=1 ;;
        h) show_help ;;
        \?) echo "[ERROR] Tùy chọn không hợp lệ: -$OPTARG" >&2; exit 1 ;;
        :)  echo "[ERROR] Tùy chọn -$OPTARG yêu cầu tham số." >&2; exit 1 ;;
    esac
done

if (( USE_LATEST == 1 )); then
    TARGET_FILE=$(find "$BACKUP_LOCAL_DIR" -name "backup_*.tar.gz" -type f -printf "%T@ %p\n" | sort -n | tail -n 1 | awk '{print $2}')
fi

if [[ -z "$TARGET_FILE" || ! -f "$TARGET_FILE" ]]; then
    log_msg "ERROR" "Không tìm thấy file backup hợp lệ để khôi phục!"
    exit 1
fi

log_msg "INFO" "=== BẮT ĐẦU QUY TRÌNH KHÔI PHỤC TOÀN DIỆN ==="
log_msg "INFO" "Tệp sao lưu nguồn: ${TARGET_FILE}"

TEMP_RESTORE_DIR="/tmp/restore_$(date '+%Y%m%d_%H%M%S')"
mkdir -p "$TEMP_RESTORE_DIR"

# 1. GIẢI NÉN GÓI BẢN SAO LƯU TỔNG
log_msg "INFO" "Đang giải nén gói sao lưu..."
tar -xzf "$TARGET_FILE" -C "$TEMP_RESTORE_DIR"
UNPACKED_DIR=$(find "$TEMP_RESTORE_DIR" -mindepth 1 -maxdepth 1 -type d | head -n 1)

# 2. KHÔI PHỤC TẦNG NGINX, WEB & TLS
if [[ -d "${UNPACKED_DIR}/web/nginx_conf" ]]; then
    log_msg "INFO" "Khôi phục cấu hình Nginx..."
    sudo rsync -a --delete "${UNPACKED_DIR}/web/nginx_conf/" /etc/nginx/
fi

if [[ -d "${UNPACKED_DIR}/web/ssl" ]]; then
    log_msg "INFO" "Khôi phục chứng chỉ TLS..."
    [[ -f "${UNPACKED_DIR}/web/ssl/lab-selfsigned.crt" ]] && sudo cp "${UNPACKED_DIR}/web/ssl/lab-selfsigned.crt" /etc/ssl/certs/ && sudo chmod 644 /etc/ssl/certs/lab-selfsigned.crt
    [[ -f "${UNPACKED_DIR}/web/ssl/lab-selfsigned.key" ]] && sudo cp "${UNPACKED_DIR}/web/ssl/lab-selfsigned.key" /etc/ssl/private/ && sudo chmod 600 /etc/ssl/private/lab-selfsigned.key
fi

sudo nginx -t && sudo systemctl reload nginx

if [[ -d "${UNPACKED_DIR}/web/www_data" ]]; then
    log_msg "INFO" "Khôi phục nội dung web /var/www..."
    sudo cp -r "${UNPACKED_DIR}/web/www_data/"* /var/www/
fi

# 3. KHÔI PHỤC TẦNG APP & SYSTEMD
if [[ -d "${UNPACKED_DIR}/app/code" ]]; then
    log_msg "INFO" "Khôi phục mã nguồn app vào /opt/myapp..."
    sudo mkdir -p /opt/myapp
    sudo cp -r "${UNPACKED_DIR}/app/code/"* /opt/myapp/
    sudo chown -R myapp:myapp /opt/myapp/
fi

if [[ -f "${UNPACKED_DIR}/app/secrets/myapp.env" ]]; then
    log_msg "INFO" "Khôi phục file secrets /etc/myapp/myapp.env..."
    sudo mkdir -p /etc/myapp
    sudo cp "${UNPACKED_DIR}/app/secrets/myapp.env" /etc/myapp/myapp.env
    sudo chown root:myapp /etc/myapp/myapp.env 2>/dev/null || sudo chown root:root /etc/myapp/myapp.env
    sudo chmod 640 /etc/myapp/myapp.env
fi

if [[ -f "${UNPACKED_DIR}/systemd/myapp.service" ]]; then
    sudo cp "${UNPACKED_DIR}/systemd/myapp.service" /etc/systemd/system/
    sudo systemctl daemon-reload
    sudo systemctl restart myapp
fi

# 4. KHÔI PHỤC TẦNG CSDL
DB_RESTORE_SCRIPT="/home/sysadmin/scripts/scripts_backup_restore_DB/db_restore.sh"
SQL_FILE=$(find "${UNPACKED_DIR}/database" -name "*.sql.gz" | head -n 1)

if [[ -f "$DB_RESTORE_SCRIPT" && -n "$SQL_FILE" && -f "$SQL_FILE" ]]; then
    log_msg "INFO" "Kích hoạt script khôi phục CSDL với file: $(basename "$SQL_FILE")..."
    sudo bash "$DB_RESTORE_SCRIPT" "$SQL_FILE"
    log_msg "INFO" "Khôi phục CSDL thành công."
else
    log_msg "WARN" "Không tìm thấy file SQL dump hoặc db_restore.sh để khôi phục CSDL."
fi

# DỌN DẸP THƯ MỤC TẠM
sudo rm -rf "$TEMP_RESTORE_DIR"

log_msg "INFO" "=== KHÔI PHỤC TOÀN BỘ HỆ THỐNG THÀNH CÔNG ==="
send_telegram_alert "♻️ *[VM1]* Hệ thống (MySQL + App + Nginx) đã được khôi phục hoàn chỉnh từ bản sao lưu \`$(basename "$TARGET_FILE")\`!"
```

---

### 3.4 Script Triển khai an toàn có Tự động Rollback (`scripts/deploy.sh`)

Triển khai cấu hình Nginx hoặc mã nguồn App; nếu `nginx -t` lỗi hoặc endpoint `/health` không trả về HTTP 200, script tự động gọi `rsync --delete` hoàn nguyên về trạng thái cũ

**Cách chạy kiểm tra cli:**
```
./home/sysadmin/ops-tool/scripts//deploy.sh --help
Sử dụng: deploy.sh [TÙY CHỌN]

Công cụ triển khai cấu hình / mã nguồn an toàn có cơ chế Auto-Rollback.

Tùy chọn:
  -s <đường_dẫn>    Thư mục chứa mã nguồn/cấu hình mới cần deploy
  -t <loại>         Loại deploy: 'nginx' hoặc 'app' (Mặc định: nginx)
  -h, --help        Hiển thị hướng dẫn này

Ví dụ:
  deploy.sh -s /path/to/new_nginx_conf -t nginx
  deploy.sh -s /path/to/new_app_code -t app
```
```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

# shellcheck source=/dev/null
source "${BASE_DIR}/config.env"
# shellcheck source=/dev/null
source "${BASE_DIR}/lib/alert.sh"

LOG_FILE="${BASE_DIR}/logs/deploy.log"
mkdir -p "$(dirname "$LOG_FILE")"

ROLLBACK_DIR="/tmp/deploy_backup_$(date '+%Y%m%d_%H%M%S')"
APP_TARGET_DIR="/opt/myapp"

log_msg() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] ${message}" | tee -a "$LOG_FILE"
}

perform_rollback() {
    trap - ERR
    log_msg "WARN" "⚠️ PHÁT HIỆN SỰ CỐ! Bắt đầu quy trình Rollback tự động..."

    if [[ -d "${ROLLBACK_DIR}/nginx" ]]; then
        sudo rsync -a --delete "${ROLLBACK_DIR}/nginx/" /etc/nginx/
        sudo systemctl reload nginx || sudo systemctl restart nginx
        log_msg "INFO" "Đã hoàn nguyên cấu hình Nginx về trạng thái an toàn."
    fi

    if [[ -d "${ROLLBACK_DIR}/app" ]]; then
        sudo rsync -a --delete "${ROLLBACK_DIR}/app/" "${APP_TARGET_DIR}/"
        sudo chown -R myapp:myapp "${APP_TARGET_DIR}"
        sudo systemctl restart myapp || true
        log_msg "INFO" "Đã hoàn nguyên mã nguồn App về phiên bản an toàn."
    fi

    sudo rm -rf "$ROLLBACK_DIR"
    local alert="🚨 [VM1] Deploy THẤT BẠI! Hệ thống đã tự động ROLLBACK về phiên bản an toàn gần nhất."
    send_telegram_alert "$alert"
    log_msg "ERROR" "Rollback hoàn tất. Đã gửi cảnh báo Telegram."
    exit 1
}

cleanup_on_err() {
    local exit_code=$?
    local line_no=$1
    log_msg "FATAL" "Deploy thất bại tại dòng ${line_no} (Exit: ${exit_code})"
    perform_rollback
}
trap 'cleanup_on_err ${LINENO}' ERR

show_help() {
    cat << HELP_EOF
Sử dụng: $(basename "$0") [TÙY CHỌN]

Công cụ triển khai cấu hình / mã nguồn an toàn có cơ chế Auto-Rollback.

Tùy chọn:
  -s <đường_dẫn>    Thư mục chứa mã nguồn/cấu hình mới cần deploy
  -t <loại>         Loại deploy: 'nginx' hoặc 'app' (Mặc định: nginx)
  -h, --help        Hiển thị hướng dẫn này

Ví dụ:
  $(basename "$0") -s /path/to/new_nginx_conf -t nginx
  $(basename "$0") -s /path/to/new_app_code -t app
HELP_EOF
    exit 0
}

SRC_DIR=""
DEPLOY_TYPE="nginx"

for arg in "$@"; do
    if [[ "$arg" == "--help" ]]; then
        show_help
    fi
done

while getopts ":s:t:h" opt; do
    case "$opt" in
        s) SRC_DIR="$OPTARG" ;;
        t) DEPLOY_TYPE="$OPTARG" ;;
        h) show_help ;;
        \?) echo "[ERROR] Tùy chọn không hợp lệ: -$OPTARG" >&2; exit 1 ;;
        :)  echo "[ERROR] Tùy chọn -$OPTARG yêu cầu tham số." >&2; exit 1 ;;
    esac
done

if [[ -z "$SRC_DIR" || ! -d "$SRC_DIR" ]]; then
    log_msg "ERROR" "Thư mục nguồn (-s) không hợp lệ hoặc không tồn tại!"
    exit 1
fi

log_msg "INFO" "=== BẮT ĐẦU QUY TRÌNH DEPLOY (${DEPLOY_TYPE^^}) ==="

# 1. TẠO BẢN BACKUP TẠM PHỤC VỤ ROLLBACK
if [[ "$DEPLOY_TYPE" == "nginx" ]]; then
    sudo mkdir -p "${ROLLBACK_DIR}/nginx"
    sudo rsync -a /etc/nginx/ "${ROLLBACK_DIR}/nginx/"

    log_msg "INFO" "Sao chép cấu hình mới vào /etc/nginx/..."
    sudo cp -r "${SRC_DIR}/"* /etc/nginx/
    sudo chown -R root:root /etc/nginx
    sudo find /etc/nginx -type f -exec chmod 644 {} \;
    sudo find /etc/nginx -type d -exec chmod 755 {} \;

    log_msg "INFO" "Kiểm tra cú pháp cấu hình (nginx -t)..."
    sudo nginx -t

    log_msg "INFO" "Tái nạp dịch vụ Nginx..."
    sudo systemctl reload nginx

    log_msg "INFO" "Xác nhận dịch vụ Nginx đang chạy..."
    sudo systemctl is-active --quiet nginx

elif [[ "$DEPLOY_TYPE" == "app" ]]; then
    sudo mkdir -p "${ROLLBACK_DIR}/app"
    sudo rsync -a "${APP_TARGET_DIR}/" "${ROLLBACK_DIR}/app/"

    log_msg "INFO" "Sao chép mã nguồn app mới vào ${APP_TARGET_DIR}..."
    sudo cp -r "${SRC_DIR}/"* "${APP_TARGET_DIR}/"
    sudo chown -R myapp:myapp "${APP_TARGET_DIR}"

    log_msg "INFO" "Khởi động lại dịch vụ myapp..."
    sudo systemctl restart myapp
    sleep 2

    log_msg "INFO" "Kiểm tra phản hồi endpoint /health..."
    curl -k -fsS -H "Host: app.lab.local" https://127.0.0.1/health > /dev/null
fi

sudo rm -rf "$ROLLBACK_DIR"
log_msg "INFO" "=== DEPLOY THÀNH CÔNG ==="
send_telegram_alert "🚀 *[VM1]* Deploy \`${DEPLOY_TYPE}\` thành công!"
```

---

### 3.5 Script Xoay vòng & Nén log (`scripts/logrotate.sh`)
Nén các file .log thành .gz có timestamp, báo hiệu cho Nginx mở lại log (reopen), và giữ đúng $N$ thế hệ log gần nhất.

**Cách chạy CLI:**
```
./home/sysadmin/ops-tool/scripts//logrotate.sh --help
Sử dụng: logrotate.sh [TÙY CHỌN]

Công cụ tự động xoay vòng, nén và dọn dẹp nhật ký hệ thống.

Tùy chọn:
  -d <đường_dẫn>    Thư mục chứa log cần xoay (Mặc định: /var/log/nginx)
  -k <số_lượng>     Số lượng thế hệ log lưu giữ (Mặc định: 5)
  -h, --help        Hiển thị hướng dẫn này

Ví dụ:
  logrotate.sh -d /var/log/nginx -k 7
```
**Scipt:**
```bash
#!/usr/bin/env bash
set -euo pipefail

# Thư mục gốc dự án
BASE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")/.." && pwd)"

# shellcheck source=/dev/null
source "${BASE_DIR}/config.env"
# shellcheck source=/dev/null
source "${BASE_DIR}/lib/alert.sh"

LOG_FILE="${BASE_DIR}/logs/logrotate.log"
mkdir -p "$(dirname "$LOG_FILE")"

# Giá trị mặc định
TARGET_DIR="/var/log/nginx"
KEEP_GENERATIONS=5

# Trap bắt lỗi
cleanup_on_err() {
    local exit_code=$?
    local line_no=$1
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [FATAL] logrotate.sh lỗi tại dòng ${line_no} (Exit: ${exit_code})" | tee -a "$LOG_FILE"
}
trap 'cleanup_on_err ${LINENO}' ERR

log_msg() {
    local level="$1"
    local message="$2"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] [${level}] ${message}" | tee -a "$LOG_FILE"
}

show_help() {
    cat << HELP_EOF
Sử dụng: $(basename "$0") [TÙY CHỌN]

Công cụ tự động xoay vòng, nén và dọn dẹp nhật ký hệ thống.

Tùy chọn:
  -d <đường_dẫn>    Thư mục chứa log cần xoay (Mặc định: /var/log/nginx)
  -k <số_lượng>     Số lượng thế hệ log lưu giữ (Mặc định: 5)
  -h, --help        Hiển thị hướng dẫn này

Ví dụ:
  $(basename "$0") -d /var/log/nginx -k 7
HELP_EOF
    exit 0
}

# Xử lý cờ --help
for arg in "$@"; do
    if [[ "$arg" == "--help" ]]; then
        show_help
    fi
done

# Phân tích tham số với getopts
while getopts ":d:k:h" opt; do
    case "$opt" in
        d) TARGET_DIR="$OPTARG" ;;
        k) KEEP_GENERATIONS="$OPTARG" ;;
        h) show_help ;;
        \?) echo "[ERROR] Tùy chọn không hợp lệ: -$OPTARG" >&2; exit 1 ;;
        :)  echo "[ERROR] Tùy chọn -$OPTARG yêu cầu tham số." >&2; exit 1 ;;
    esac
done

log_msg "INFO" "=== BẮT ĐẦU XOAY VÒNG LOG ==="
log_msg "INFO" "Thư mục mục tiêu: ${TARGET_DIR} | Số thế hệ lưu: ${KEEP_GENERATIONS}"

if [[ ! -d "$TARGET_DIR" ]]; then
    log_msg "ERROR" "Thư mục ${TARGET_DIR} không tồn tại!"
    exit 1
fi

TIMESTAMP="$(date '+%Y%m%d_%H%M%S')"

# 1. Đổi tên và nén các file .log hiện tại
for raw_log in "${TARGET_DIR}"/*.log; do
    [[ -e "$raw_log" ]] || continue

    filename="$(basename "$raw_log")"
    rotated_name="${raw_log}.${TIMESTAMP}"

    # Đổi tên file log đang chạy
    sudo mv "$raw_log" "$rotated_name"
    # Nén tệp thành định dạng .gz
    sudo gzip "$rotated_name"
    log_msg "INFO" "Đã xoay và nén: ${filename} -> ${filename}.${TIMESTAMP}.gz"
done

# 2. Báo hiệu dịch vụ ghi vào file log mới
if systemctl is-active --quiet nginx; then
    sudo nginx -s reopen || sudo systemctl reload nginx
    log_msg "INFO" "Đã gửi tín hiệu reopen log tới Nginx."
fi

# 3. Dọn dẹp các bản log cũ vượt quá số lượng N thế hệ
# Nhóm theo từng loại log (ví dụ: access.log.*.gz, error.log.*.gz)
for base_file in "${TARGET_DIR}"/*.log; do
    base_name="$(basename "$base_file")"

    # Tìm toàn bộ các file .gz tương ứng, sắp xếp theo thời gian mới nhất
    mapfile -t old_logs < <(find "$TARGET_DIR" -maxdepth 1 -name "${base_name}.*.gz" -type f -printf "%T@ %p\n" | sort -nr | awk '{print $2}')

    total_files="${#old_logs[@]}"
    if (( total_files > KEEP_GENERATIONS )); then
        files_to_delete=( "${old_logs[@]:KEEP_GENERATIONS}" )
        for file in "${files_to_delete[@]}"; do
            sudo rm -f "$file"
            log_msg "INFO" "Đã dọn dẹp log cũ vượt mốc: $(basename "$file")"
        done
    fi
done

log_msg "INFO" "=== HOÀN TẤT XOAY VÒNG LOG ==="
```

---

## 4. Menu CLI Điều phối Tương tác (`main.sh`)



Là điểm vào duy nhất để tương tác toàn bộ hệ thống bằng giao diện dòng lệnh.

**Cách chạy:**

```bash
cd /home/sysadmin/ops-tool
./main.sh
```

```bash
#!/usr/bin/env bash
set -euo pipefail

BASE_DIR="$(cd "$(dirname "${BASH_SOURCE[0]}")" && pwd)"

# shellcheck source=/dev/null
source "${BASE_DIR}/config.env"
# shellcheck source=/dev/null
source "${BASE_DIR}/lib/alert.sh"

show_menu() {
    clear
    echo "======================================================"
    echo "   HỆ THỐNG VẬN HÀNH & TỰ ĐỘNG HÓA LINUX (VM1)        "
    echo "======================================================"
    echo " 1) Kiểm tra sức khỏe hệ thống (Health-check)"
    echo " 2) Sao lưu toàn diện (Backup & Rsync VM2)"
    echo " 3) Khôi phục hệ thống (Restore từ bản mới nhất)"
    echo " 4) Xoay vòng và nén nhật ký (Logrotate)"
    echo " 5) Triển khai thử nghiệm (Deploy Nginx/App)"
    echo " 6) Gửi cảnh báo thử nghiệm tới Telegram"
    echo " 7) Kiểm tra cú pháp toàn bộ Script (ShellCheck)"
    echo " 0) Thoát (Exit)"
    echo "------------------------------------------------------"
}

pause() {
    echo ""
    read -rp "Nhấn [Enter] để quay lại Menu..."
}

while true; do
    show_menu
    read -rp "Vui lòng chọn chức năng [0-7]: " choice

    case "$choice" in
        1)
            echo -e "\n>>> Đang chạy Health-check..."
            "${BASE_DIR}/scripts/healthcheck.sh" || true
            pause
            ;;
        2)
            echo -e "\n>>> Đang chạy Backup & Rsync sang VM2..."
            "${BASE_DIR}/scripts/backup.sh" || true
            pause
            ;;
        3)
            echo -e "\n>>> Đang khôi phục từ bản sao lưu gần nhất..."
            "${BASE_DIR}/scripts/restore.sh" -l || true
            pause
            ;;
        4)
            echo -e "\n>>> Đang xoay vòng log Nginx..."
            "${BASE_DIR}/scripts/logrotate.sh" -d /var/log/nginx -k 5 || true
            pause
            ;;
        5)
            echo -e "\n>>> TRIỂN KHAI AN TOÀN (DEPLOY)"
            echo "Gợi ý thư mục test: "
            echo " - Lỗi Nginx: ${BASE_DIR}/nginx_temp/nginx_wrong"
            echo " - Đúng Nginx: ${BASE_DIR}/nginx_temp/nginx_right"
            read -rp "Nhập đường dẫn thư mục nguồn (-s): " custom_path
            read -rp "Nhập loại deploy ('nginx' hoặc 'app'): " dep_type
            if [[ -d "$custom_path" ]]; then
                "${BASE_DIR}/scripts/deploy.sh" -s "$custom_path" -t "$dep_type" || true
            else
                echo "[ERROR] Thư mục không tồn tại!"
            fi
            pause
            ;;
        6)
            echo -e "\n>>> Đang gửi tin nhắn thử nghiệm tới Telegram..."
            send_telegram_alert "🔔 *[VM1]* Test cảnh báo thủ công từ Menu CLI vận hành!"
            echo "[OK] Đã gửi tin nhắn."
            pause
            ;;
        7)
            echo -e "\n>>> Đang kiểm tra mã nguồn bằng ShellCheck..."
            if command -v shellcheck >/dev/null 2>&1; then
                shellcheck "${BASE_DIR}/main.sh" "${BASE_DIR}"/scripts/*.sh "${BASE_DIR}"/lib/*.sh && echo "✅ Tất cả script đều đạt chuẩn ShellCheck 100% sạch lỗi!" || true
            else
                echo "[!] Chưa cài đặt shellcheck. Chạy 'sudo apt-get install -y shellcheck' để cài đặt."
            fi
            pause
            ;;
        0)
            echo -e "\nThoát khỏi chương trình. Tạm biệt!"
            exit 0
            ;;
        *)
            echo -e "\n[Lỗi] Lựa chọn không hợp lệ! Vui lòng chỉ nhập số từ 0 đến 7."
            sleep 2
            ;;
    esac
done
```

---

## 5. Đặt lịch tự động hóa với `cron` (trên VM1)



Thiết lập crontab cho user `sysadmin`:

```bash
crontab -e
```

Thêm khối lịch trình vận hành:

```cron
TZ=Asia/Ho_Chi_Minh

# 1. Health-check hệ thống mỗi 5 phút
*/5 * * * * /home/sysadmin/ops-tool/scripts/healthcheck.sh >/dev/null 2>&1

# 2. Sao lưu toàn diện & đồng bộ sang VM2 lúc 02:00 sáng mỗi ngày
0 2 * * * /home/sysadmin/ops-tool/scripts/backup.sh >/dev/null 2>&1

# 3. Xoay vòng log Nginx lúc 00:00 Chủ Nhật hàng tuần
0 0 * * 0 /home/sysadmin/ops-tool/scripts/logrotate.sh -d /var/log/nginx -k 5 >/dev/null 2>&1
```

---

## 6. Kịch bản kiểm thử & Demo trực tiếp trước Hội đồng



### 6.1 Kiểm tra ShellCheck trực tiếp sạch lỗi 100%



```bash
shellcheck /home/sysadmin/ops-tool/main.sh /home/sysadmin/ops-tool/scripts/*.sh /home/sysadmin/ops-tool/lib/*.sh
```

(Kỳ vọng: Lệnh thực thi thành công, không xuất hiện dòng lỗi hoặc cảnh báo nào).

---

### 6.2 Mô phỏng sự cố để kích hoạt Cảnh báo Telegram



* **Trường hợp 1: Dừng dịch vụ `myapp`**

```bash
sudo systemctl stop myapp
/home/sysadmin/ops-tool/scripts/healthcheck.sh
# Kiểm tra tin nhắn báo động đỏ trên điện thoại
sudo systemctl start myapp
```


* **Trường hợp 2: Tải cao CPU vượt ngưỡng 80%**
```bash
# Kích hoạt stress CPU chạy nền trong 30 giây
stress-ng --cpu 0 --timeout 30s >/dev/null 2>&1 &
/home/sysadmin/ops-tool/scripts/healthcheck.sh
```



---

### 6.3 Demo Triển khai Nginx Lỗi -> Tự động Rollback (Dùng `nginx_temp`)



* Bước 1: Chạy deploy thư mục cấu hình lỗi (`nginx_wrong`)


```bash
./home/sysadmin/ops-tool/scripts/deploy.sh -s /home/sysadmin/ops-tool/nginx_temp/nginx_wrong -t nginx
```


* **Giải thích:** `deploy.sh` sao lưu `/etc/nginx` ra `/tmp/deploy_backup_*`, nạp file `nginx.conf` chứa dòng lỗi `abcdef`, chạy `nginx -t` phát hiện lỗi $\rightarrow$ `trap ERR` gọi `perform_rollback` dùng `rsync --delete` khôi phục ngay trạng thái cũ và gửi cảnh báo Telegram.


* **Kỳ vọng:** Nginx không bị sập, `sudo systemctl status nginx` vẫn `active (running)`.


* Bước 2: Chạy deploy thư mục cấu hình chuẩn (`nginx_right`)


```bash
./home/sysadmin/ops-tool/scripts/deploy.sh -s /home/sysadmin/ops-tool/nginx_temp/nginx_right -t nginx
```


* **Kỳ vọng:** `nginx -t` hợp lệ, thực hiện `systemctl reload nginx`, Telegram nhận tin nhắn `Deploy nginx thành công!`.



---

### 6.4 Sao lưu toàn diện & Phá hủy CSDL -> Khôi phục trực tiếp



```bash
# 1. Ghi 1 bản ghi mới vào CSDL qua API
curl -k -X POST -H "Host: app.lab.local" -H "Content-Type: application/json" \
     -d '{"name":"DEMO_FINAL_GRADE_10"}' [https://127.0.0.1/api/items](https://127.0.0.1/api/items)

# 2. Chạy sao lưu toàn diện & đẩy sang VM2
./home/sysadmin/ops-tool/scripts/backup.sh

# 3. Chứng minh file .tar.gz đã đồng bộ sang VM2
ssh vmbackup@10.10.10.12 "ls -lh ~/backups/"

# 4. Phá hủy bảng dữ liệu
sudo mysql -u root app_db -e "DROP TABLE items;"
curl -k -H "Host: app.lab.local" [https://127.0.0.1/api/items](https://127.0.0.1/api/items)  # Báo lỗi

# 5. Khôi phục thảm họa từ bản lưu mới nhất
./home/sysadmin/ops-tool/scripts/restore.sh -l

# 6. Chứng minh dữ liệu phục hồi 100%
curl -k -H "Host: app.lab.local" [https://127.0.0.1/api/items](https://127.0.0.1/api/items) | python3 -m json.tool
```

---

## 7. Lệnh kiểm tra tổng hợp trước buổi Demo



```bash
shellcheck /home/sysadmin/ops-tool/main.sh /home/sysadmin/ops-tool/scripts/*.sh /home/sysadmin/ops-tool/lib/*.sh
ls -la /data/backups/
ssh vmbackup@10.10.10.12 "ls -la ~/backups/"
curl -k -H "Host: app.lab.local" [https://127.0.0.1/health](https://127.0.0.1/health)
crontab -l
```

---

## Ghi chú quan trọng cho báo cáo kỹ thuật (Mục Nhìn lại & Xử lý sự cố)



1. **Sự cố `LOCK TABLES` khi sao lưu CSDL (Error 1044):** Tài khoản `app_user` được phân quyền theo nguyên tắc đặc quyền tối thiểu (`SELECT, INSERT, UPDATE, DELETE`). Khi `mysqldump` chạy mặc định sẽ cố khóa bảng, gây lỗi phân quyền. Khắc phục bằng cách thêm cờ `--single-transaction --skip-lock-tables` vào lệnh dump để tạo transaction snapshot an toàn mà không làm gián đoạn ứng dụng.


2. **Sự cố Phân quyền khi giải nén phục hồi CSDL (Error 1142):** Script `db_restore.sh` ban đầu dùng `app_user` để nạp dữ liệu nhưng bị chặn do file dump chứa lệnh `DROP TABLE IF EXISTS`. Khắc phục: Khôi phục thảm họa là tác vụ của quản trị viên (Root/DBA), do đó lệnh restore được chuyển sang thực thi dưới quyền `sudo mysql`.


3. **Sự cố Đóng gói file bảo mật `myapp.env` (Permission Denied):** File bí mật CSDL có quyền `640` (`root:myapp`) và TLS key có quyền `600` (`root:root`). Lệnh `tar` chạy bằng user thường không thể đọc. Khắc phục: Chạy `sudo tar -czf` và dùng `sudo chown sysadmin:sysadmin` trên file archive nén để lệnh `rsync` truyền file an toàn sang VM2.


4. **Đồng bộ đường dẫn thực thi Production `/opt/myapp`:** Ban đầu script trỏ nhầm vào workspace `~/project02/app`. Khắc phục: Trỏ toàn bộ tiến trình Deploy, Backup và Restore vào đúng thư mục vận hành chính thức `/opt/myapp` với quyền sở hữu chuẩn `myapp:myapp` theo khai báo của `myapp.service`.
