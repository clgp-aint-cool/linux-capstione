# Tổng hợp lệnh kiểm tra — Web Server & Thành phần 2

---

## 1. Kiểm tra dịch vụ đang chạy (tổng quan)

```bash
sudo systemctl status nginx
sudo systemctl status mysql
sudo systemctl status myapp
```

Kỳ vọng cả 3 đều: `Active: active (running)`, `Loaded: ... enabled`.

---

## 2. Kiểm tra các cổng đang lắng nghe

```bash
sudo ss -tlnp | grep -E "80|443|3306|8000"
```

Kỳ vọng:

| Cổng | Địa chỉ | Dịch vụ | Ý nghĩa |
|---|---|---|---|
| 80 | `0.0.0.0` hoặc `*` | nginx | HTTP — chỉ dùng để redirect sang HTTPS |
| 443 | `0.0.0.0` hoặc `*` | nginx | HTTPS — phục vụ app thật |
| 8000 | `127.0.0.1` | gunicorn | App nội bộ, KHÔNG lộ ra ngoài, chỉ Nginx gọi tới |
| 3306 | `127.0.0.1` | mysqld | DB chỉ nghe localhost — **không được** thấy `0.0.0.0:3306` |

---

## 3. Kiểm tra cấu hình Nginx (virtual host)

```bash
sudo nginx -t
ls -la /etc/nginx/sites-enabled/
```

Kỳ vọng: cú pháp OK, có ≥2 vhost (`app.conf` + `status.conf` hoặc tương đương).

---

## 4. Kiểm tra tường lửa

```bash
sudo ufw status
```

Kỳ vọng thấy `22/tcp`, `80/tcp`, `443/tcp` đều `ALLOW`, các cổng khác mặc định chặn.

---

## 5. Test HTTP → HTTPS redirect

```bash
curl -I -H "Host: app.lab.local" http://127.0.0.1/
```

Kỳ vọng: `HTTP/1.1 301 Moved Permanently` + `Location: https://app.lab.local/`

---

## 6. Test app qua HTTPS — endpoint health

```bash
curl -k -H "Host: app.lab.local" https://127.0.0.1/health
```

Kỳ vọng: `{"db":"reachable","status":"ok"}`

---

## 7. Test app qua HTTPS — đọc dữ liệu (GET)

```bash
curl -k -H "Host: app.lab.local" https://127.0.0.1/api/items | python3 -m json.tool
```

Kỳ vọng: trả về danh sách item dạng JSON.

---

## 8. Test app qua HTTPS — ghi dữ liệu (POST)

```bash
curl -k -X POST -H "Host: app.lab.local" -H "Content-Type: application/json" \
     -d '{"name":"test demo"}' https://127.0.0.1/api/items
```

Kỳ vọng: trả về item vừa tạo kèm `id` mới.

---

## 9. Kiểm tra MySQL — DB đặc quyền tối thiểu

```bash
sudo mysql -u root -e "SHOW GRANTS FOR 'app_user'@'localhost';"
```

Kỳ vọng: chỉ `SELECT, INSERT, UPDATE, DELETE` — **không** có `ALL PRIVILEGES`.

```bash
sudo mysql -u root app_db -e "SHOW TABLES; SELECT * FROM items;"
```

Kỳ vọng: bảng `items` tồn tại, có dữ liệu.

---

## 10. Chứng minh đặc quyền tối thiểu — thử lệnh bị từ chối

```bash
mysql -u app_user -p -h 127.0.0.1 app_db -e "DROP TABLE items;"
mysql -u app_user -p -h 127.0.0.1 app_db -e "CREATE TABLE test_table (id INT);"
```

Kỳ vọng cả hai đều báo lỗi:
```
ERROR 1142 (42000): DROP command denied to user 'app_user'@'localhost' ...
ERROR 1142 (42000): CREATE command denied to user 'app_user'@'localhost' ...
```

---

## 11. Kiểm tra file secrets

```bash
ls -l /etc/myapp/myapp.env
```

Kỳ vọng: `-rw-r----- 1 root myapp ...` (chmod 640, chủ root:myapp)

```bash
sudo grep -c "^DB_PASSWORD=" /etc/myapp/myapp.env    # phải ra 1
sudo grep "DB_PASSWORD=<" /etc/myapp/myapp.env       # phải KHÔNG ra gì (không còn placeholder)
```

---

## 12. Demo bắt buộc — kill process, systemd tự khởi động lại

```bash
sudo systemctl status myapp | grep Main
sudo kill -9 <PID_thấy_ở_trên>
sleep 2
sudo systemctl status myapp                          # phải thấy active lại, PID mới
sudo journalctl -u myapp --since "2 min ago" | tail -20
```

Kỳ vọng thấy trong log:
```
Main process exited, code=killed, status=9/KILL
Scheduled restart job, restart counter is at N
Started MyApp Flask service...
```

---

## 13. Kiểm tra chứng chỉ TLS

```bash
sudo ls -l /etc/ssl/private/lab-selfsigned.key /etc/ssl/certs/lab-selfsigned.crt
```

Kỳ vọng: `.key` chmod `600` (chỉ root đọc), `.crt` chmod `644` (đọc công khai được).

---

## 14. Kiểm tra service account `myapp` không login được

```bash
sudo su - myapp
```

Kỳ vọng: `This account is currently not available.`

---

## Checklist nhanh trước demo (đánh dấu khi xong)

- [ ] `systemctl status nginx/mysql/myapp` — cả 3 active + enabled
- [ ] `ss -tlnp` — 80/443 public, 3306/8000 chỉ localhost
- [ ] `ufw status` — chỉ mở đúng cổng cần
- [ ] `curl -I http://` — redirect 301 sang HTTPS
- [ ] `curl -k https://.../health` — db reachable
- [ ] `curl -k https://.../api/items` GET + POST — hoạt động
- [ ] `SHOW GRANTS` — app_user không có ALL PRIVILEGES
- [ ] `DROP TABLE` / `CREATE TABLE` bằng app_user — bị từ chối
- [ ] `/etc/myapp/myapp.env` — chmod 640, không còn placeholder
- [ ] kill -9 process app — systemd tự restart, log rõ ràng
- [ ] chứng chỉ TLS — quyền file đúng
- [ ] `su - myapp` — không login được
