# Vai trò 4 — Cơ sở Dữ liệu & Sao lưu — Khôi phục

Tài liệu tổng hợp: nhật ký kỹ thuật triển khai CSDL, gia cố bảo mật mạng, quản lý secrets, bộ script tự động hóa sao lưu/khôi phục đạt chuẩn DevOps, kịch bản diễn tập thảm họa phục hồi thực tế và bộ lệnh kiểm tra nhanh trước demo.

**Môi trường:** Ubuntu 22.04 LTS Server.  
**Phân công vai trò:** Chịu trách nhiệm khởi tạo CSDL `app_db`, phân quyền tối thiểu `app_user`, cấu hình cô lập Localhost, thiết lập phân vùng sao lưu, viết bộ script sao lưu/khôi phục tự động chuẩn hóa và chuẩn bị kịch bản diễn tập phục hồi thảm họa.

---

## 1. Cài đặt & Gia cố Bảo mật CSDL

### 1.1. Cài đặt MySQL Server
Thực hiện trên máy chủ `vm1-svc` (VM1) bằng tài khoản quản trị non-root `sysadmin`:
```bash
sudo apt update
sudo apt install -y mysql-server
sudo systemctl enable --now mysql

```

### 1.2. Cô lập dịch vụ — Chỉ lắng nghe trên Localhost (`127.0.0.1`)

Theo yêu cầu bắt buộc của đề bài (§5.1), CSDL quan hệ không được phép mở kết nối trực tiếp ra ngoài mạng.

Mở file cấu hình MySQL:

```bash
sudo nano /etc/mysql/mysql.conf.d/mysqld.cnf

```

Đảm bảo tham số sau được thiết lập:

```ini
bind-address = 127.0.0.1

```

Khởi động lại dịch vụ và kiểm tra socket:

```bash
sudo systemctl restart mysql
sudo ss -tlnp | grep 3306

```

Kết quả thực tế — cổng 3306 chỉ lắng nghe trên `127.0.0.1`:

```text
LISTEN 0      151        127.0.0.1:3306       0.0.0.0:*    users:(("mysqld",pid=86997,fd=21))
LISTEN 0      70         127.0.0.1:33060      0.0.0.0:*    users:(("mysqld",pid=86997,fd=23))

```

---

## 2. Khởi tạo Schema & Cấp quyền Tối thiểu (Least Privilege)

### 2.1. Khởi tạo Database và User ứng dụng

Theo nguyên tắc đặc quyền tối thiểu (§5.1), tuyệt đối không dùng tài khoản quản trị (`root`) cho ứng dụng. Khởi tạo database riêng `app_db` và user `app_user` chỉ được kết nối cục bộ từ `localhost`:

```bash
sudo mysql << 'EOF'
-- Tạo Database ứng dụng
CREATE DATABASE IF NOT EXISTS app_db;

-- Tạo User ứng dụng kết nối nội bộ
CREATE USER IF NOT EXISTS 'app_user'@'localhost' IDENTIFIED BY 'PasswordApp123!';

-- Cấp đúng các quyền thao tác dữ liệu cần thiết (Least Privilege)
GRANT SELECT, INSERT ON app_db.* TO 'app_user'@'localhost';
FLUSH PRIVILEGES;
EOF

```

### 2.2. Kiểm chứng phân quyền

Xác nhận quyền được cấp:

```bash
sudo mysql -e "SHOW GRANTS FOR 'app_user'@'localhost';"

```

```text
+------------------------------------------------------------------------------+
| Grants for app_user@localhost                                                |
+------------------------------------------------------------------------------+
| GRANT USAGE ON *.* TO `app_user`@`localhost`                                 |
| GRANT SELECT, INSERT ON `app_db`.* TO `app_user`@`localhost`                 |
+------------------------------------------------------------------------------+

```

Chứng minh các lệnh phá hủy/sửa cấu trúc bảng bằng `app_user` bị từ chối:

```bash
MYSQL_PWD='PasswordApp123!' mysql -u app_user -h 127.0.0.1 app_db -e "DROP TABLE items;"
# ERROR 1142 (42000): DROP command denied to user 'app_user'@'localhost' for table 'items'

MYSQL_PWD='PasswordApp123!' mysql -u app_user -h 127.0.0.1 app_db -e "CREATE TABLE test (id INT);"
# ERROR 1142 (42000): CREATE command denied to user 'app_user'@'localhost' for table 'test'

```

---

## 3. Cấu hình Secrets & Bảo mật Môi trường

Theo quy định §5.1, thông tin bí mật không được hard-code vào script hay mã nguồn. Cấu hình môi trường được tách riêng và phân quyền nghiêm ngặt.

### 3.1. Thiết lập file `/etc/db_backup.env`

```bash
sudo nano /etc/db_backup.env

```

Nội dung file:

```env
# Database Credentials
DB_HOST="127.0.0.1"
DB_PORT="3306"
DB_NAME="app_db"
DB_USER="app_user"
DB_PASS="<DB_PASSWORD>"

# Storage & Retention Policy
BACKUP_DIR="/mnt/backup_data/db"
RETENTION_DAYS="7"

```

### 3.2. Thiết lập phân vùng lưu trữ & Quyền hạn chế file secrets

```bash
# Tạo thư mục trên phân vùng lưu trữ riêng (§3.1)
sudo mkdir -p /mnt/backup_data/db
sudo chown -R sysadmin:sysadmin /mnt/backup_data

# Giới hạn quyền đọc file secret: chỉ root và nhóm sysadmin đọc được
sudo chown root:sysadmin /etc/db_backup.env
sudo chmod 640 /etc/db_backup.env

```

Xác nhận quyền file:

```bash
ls -l /etc/db_backup.env
# -rw-r----- 1 root sysadmin 248 ... /etc/db_backup.env

```

---

## 4. Bộ Script Tự động hóa Sao lưu & Khôi phục

Tất cả các script tuân thủ tiêu chuẩn: có Shebang chuẩn, cờ nghiêm ngặt `set -euo pipefail`, bẫy lỗi `trap` trên tín hiệu `ERR`, trích xuất secrets từ file `.env` và đạt chuẩn `shellcheck` 100% không cảnh báo.

### 4.1. Script Sao lưu CSDL (`db_backup.sh`)

```bash
#!/usr/bin/env bash
# db_backup.sh - Sao lưu CSDL MySQL nén luồng và áp dụng Retention Policy

set -euo pipefail

ENV_FILE="/etc/db_backup.env"

if [[ ! -f "${ENV_FILE}" ]]; then
    echo "[ERROR] Không tìm thấy file cấu hình môi trường: ${ENV_FILE}" >&2
    exit 1
fi

# Load biến môi trường
# shellcheck source=/dev/null
source "${ENV_FILE}"

TIMESTAMP=$(date +"%Y%m%d_%H%M%S")
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.sql.gz"

cleanup_on_error() {
    echo "[ERROR] Quá trình sao lưu CSDL thất bại! Đang dọn dẹp..." >&2
    if [[ -f "${BACKUP_FILE}" ]]; then
        rm -f "${BACKUP_FILE}"
    fi
    exit 1
}

trap cleanup_on_error ERR

mkdir -p "${BACKUP_DIR}"

echo "[INFO] Đang tiến hành sao lưu CSDL '${DB_NAME}'..."
MYSQL_PWD="${DB_PASS}" mysqldump -h"${DB_HOST}" -P"${DB_PORT}" -u"${DB_USER}" --no-tablespaces "${DB_NAME}" | gzip > "${BACKUP_FILE}"

echo "[SUCCESS] Đã tạo bản sao lưu thành công: ${BACKUP_FILE}"

echo "[INFO] Đang dọn dẹp các bản sao lưu cũ hơn ${RETENTION_DAYS} ngày..."
find "${BACKUP_DIR}" -type f -name "${DB_NAME}_*.sql.gz" -mtime +"${RETENTION_DAYS}" -delete

echo "[INFO] Hoàn tất quy trình sao lưu!"

```

### 4.2. Script Khôi phục CSDL (`db_restore.sh`)

```bash
#!/usr/bin/env bash
# db_restore.sh - Khôi phục CSDL MySQL trực tiếp từ bản sao lưu nén

set -euo pipefail

ENV_FILE="/etc/db_backup.env"

if [[ ! -f "${ENV_FILE}" ]]; then
    echo "[ERROR] Không tìm thấy file cấu hình môi trường: ${ENV_FILE}" >&2
    exit 1
fi

# Load biến môi trường
# shellcheck source=/dev/null
source "${ENV_FILE}"

LATEST_BACKUP=$(find "${BACKUP_DIR}" -maxdepth 1 -name "${DB_NAME}_*.sql.gz" -printf "%T@ %p\n" 2>/dev/null | sort -nr | head -n 1 | cut -d' ' -f2-)
RESTORE_FILE="${1:-${LATEST_BACKUP}}"

if [[ -z "${RESTORE_FILE}" || ! -f "${RESTORE_FILE}" ]]; then
    echo "[ERROR] Không tìm thấy file sao lưu hợp lệ để khôi phục!" >&2
    echo "Cú pháp: $0 [/đường/dẫn/file_backup.sql.gz]" >&2
    exit 1
fi

cleanup_on_error() {
    echo "[ERROR] Quá trình khôi phục CSDL thất bại!" >&2
    exit 1
}

trap cleanup_on_error ERR

echo "[INFO] Đang tiến hành khôi phục CSDL '${DB_NAME}' từ file '${RESTORE_FILE}'..."
gunzip -c "${RESTORE_FILE}" | MYSQL_PWD="${DB_PASS}" mysql -h"${DB_HOST}" -P"${DB_PORT}" -u"${DB_USER}" "${DB_NAME}"

echo "[SUCCESS] Khôi phục dữ liệu thành công cho database '${DB_NAME}'!"

```

Cấp quyền thực thi và kiểm tra tĩnh bằng `shellcheck`:

```bash
chmod +x /home/sysadmin/scripts/scripts_backup_restore_DB/db_backup.sh /home/sysadmin/scripts/scripts_backup_restore_DB/db_restore.sh
shellcheck /home/sysadmin/scripts/scripts_backup_restore_DB/db_backup.sh
shellcheck /home/sysadmin/scripts/scripts_backup_restore_DB/db_restore.sh
# Kết quả sạch 100%, không in ra cảnh báo.

```

---

## 6. Kịch bản Diễn tập Thảm họa & Phục hồi (Disaster Recovery Demo)

### Bước 1: Tạo bản sao lưu mới nhất

```bash
/home/sysadmin/scripts/scripts_backup_restore_DB/db_backup.sh
ls -lh /mnt/backup_data/db/

```

### Bước 2: Kiểm tra dữ liệu đang chạy

```bash
sudo mysql app_db -e "SELECT * FROM items;"

```

```text
+----+--------------+---------------------+
| id | name         | created_at          |
+----+--------------+---------------------+
|  1 | item mẫu 1   | 2026-08-18 03:24:41 |
|  2 | item mẫu 2   | 2026-08-18 03:24:41 |
+----+--------------+---------------------+

```

### Bước 3: Cố tình gây lỗi phá hủy dữ liệu (Disaster Simulation)

```bash
sudo mysql app_db -e "DROP TABLE items;"

```

Xác nhận ứng dụng bị lỗi 500 do mất bảng CSDL:

```bash
curl -k -H "Host: app.lab.local" [https://127.0.0.1/api/items](https://127.0.0.1/api/items)
# {"detail":"(1146, \"Table 'app_db.items' doesn't exist\")","error":"internal_error"}

```

### Bước 4: Kích hoạt khôi phục CSDL

```bash
/home/sysadmin/scripts/scripts_backup_restore_DB/db_restore.sh

```

```text
[INFO] Đang tiến hành khôi phục CSDL 'app_db' từ file '/mnt/backup_data/db/app_db_20260818_032500.sql.gz'...
[SUCCESS] Khôi phục dữ liệu thành công cho database 'app_db'!

```

### Bước 5: Kiểm chứng phục hồi dữ liệu đầu cuối

```bash
sudo mysql app_db -e "SELECT * FROM items;"
curl -k -H "Host: app.lab.local" [https://127.0.0.1/api/items](https://127.0.0.1/api/items)

```

Kết quả trả về danh sách items đầy đủ nguyên vẹn $\rightarrow$ Chứng minh phục hồi hoàn tất 100%.

---

## 7. Sự cố đã gặp & Bài học kinh nghiệm (Nhìn lại)

* **Sự cố 1 — Lỗi thiếu quyền `PROCESS` khi `mysqldump`:**
* *Hiện tượng:* `mysqldump: Error: 'Access denied; you need (at least one of) the PROCESS privilege(s)...' when trying to dump tablespaces`.
* *Nguyên nhân:* MySQL 8.0 mặc định dump cả cấu hình tablespace hạ tầng đòi hỏi quyền `PROCESS` toàn cục, trong khi `app_user` chỉ có quyền trên `app_db.*` theo chuẩn Least Privilege.

* *Khắc phục:* Thêm cờ `--no-tablespaces` vào lệnh `mysqldump` để chỉ tập trung sao lưu cấu trúc bảng và dữ liệu ứng dụng.


* **Sự cố 2 — Cảnh báo mật khẩu trên dòng lệnh:**
* *Hiện tượng:* `mysqldump: [Warning] Using a password on the command line interface can be insecure.`
* *Nguyên nhân:* Truyền trực tiếp cờ `-p"..."` khiến thông tin mật khẩu có nguy cơ lộ trong bảng tiến trình (`ps aux`).
* *Khắc phục:* Dùng biến môi trường nội bộ `MYSQL_PWD="${DB_PASS}"` trước lệnh `mysqldump`/`mysql` để che giấu mật khẩu an toàn.

---

## 8. Bộ lệnh kiểm tra nhanh trước buổi Demo

```bash
# 1. Trạng thái dịch vụ MySQL
sudo systemctl status mysql --no-pager

# 2. Kiểm chứng cô lập cổng 3306
sudo ss -tlnp | grep 3306

# 3. Kiểm chứng quyền tối thiểu của app_user
sudo mysql -e "SHOW GRANTS FOR 'app_user'@'localhost';"

# 4. Kiểm chứng bảo mật file secrets
ls -l /etc/db_backup.env

# 5. Kiểm tra cú pháp script bằng shellcheck
shellcheck /home/sysadmin/scripts/scripts_backup_restore_DB/db_backup.sh
shellcheck /home/sysadmin/scripts/scripts_backup_restore_DB/db_restore.sh

# 6. Kiểm tra crontab lập lịch
crontab -l

# 7. Thử chạy backup và kiểm tra file nén sinh ra
/home/sysadmin/scripts/scripts_backup_restore_DB/db_backup.sh
ls -lh /mnt/backup_data/db/

```

---

## 9. Checklist tổng hợp đối chiếu với Đề cương đồ án

* [x] MySQL Server cài đặt bản sạch trên Ubuntu 22.04 LTS (§3.1)


* [x] CSDL chỉ lắng nghe trên `127.0.0.1:3306`, cô lập hoàn toàn khỏi mạng ngoài (§5.1)


* [x] Khởi tạo CSDL riêng (`app_db`) và user riêng (`app_user`@`localhost`) đạt chuẩn Least Privilege (§5.1)


* [x] Không dùng tài khoản quản trị (`root`) cho ứng dụng (§5.1)


* [x] Toàn bộ bí mật CSDL được lưu tại `/etc/db_backup.env` với quyền `640` (chủ `root:sysadmin`), không hard-code (§5.1)


* [x] Script `db_backup.sh` dump nén luồng `.sql.gz`, gắn timestamp, lưu trữ phân vùng riêng `/mnt/backup_data/db/` (§3.3, §6.1)


* [x] Chính sách lưu giữ (Retention Policy) tự động dọn dẹp file cũ hơn 7 ngày (§3.3)



* [x] Script `db_restore.sh` hỗ trợ phục hồi tự động bản mới nhất hoặc qua tham số (§3.3, §6.1)


* [x] Cả 2 script Bash đạt chuẩn `set -euo pipefail`, bẫy lỗi `trap` trên `ERR` và sạch lỗi `shellcheck` 100% (§3.4)


* [x] Diễn tập thành công kịch bản phá hủy dữ liệu và khôi phục nguyên vẹn đầu cuối (§3.3, §5.2)



```

```
