(1) hạ tầng & mạng -- Phu

(2) reverse proxy & bảo mật web -- Phuc

(3) ứng dụng & systemd -- Duy

(4)CSDL & sao lưu -- Khanh

(5) bộ công cụ tự động hóa & cảnh báo -- Long

## Checklist Yêu cầu

### Hệ thống & lưu trữ

| ✓   | Yêu cầu                                                                 | Áp dụng | Tiêu chí |
| --- | ----------------------------------------------------------------------- | ------- | -------- |
| [ ] | Cài Ubuntu 22.04 hoặc CentOS Stream 9 bản sạch; đặt hostname có ý nghĩa | Tất cả  | Hệ thống |
| [ ] | Dùng tài khoản quản trị non-root (sudo) cho công việc hằng ngày         | Tất cả  | Bảo mật  |
| [ ] | Vùng dữ liệu/sao lưu riêng mount qua `/etc/fstab`, tồn tại sau reboot   | Tất cả  | Hệ thống |
| [ ] | Kiến trúc từ 2 VM trở lên (bắt buộc với nhóm 5 người)                   | Tất cả  | Hệ thống |

### Gia cố bảo mật

| ✓   | Yêu cầu                                                                 | Áp dụng | Tiêu chí |
| --- | ----------------------------------------------------------------------- | ------- | -------- |
| [ ] | Tường lửa (UFW hoặc firewalld) mặc định chặn vào; chỉ mở cổng cần thiết | Tất cả  | Bảo mật  |
| [ ] | Gia cố SSH: tắt root, xác thực khóa, đổi cổng hoặc AllowUsers           | Tất cả  | Bảo mật  |
| [ ] | fail2ban bảo vệ SSH; ban được kích hoạt trực tiếp                       | Tất cả  | Bảo mật  |
| [ ] | auditd theo dõi `/etc/passwd` và `/etc/shadow`; dấu vết đọc được        | Tất cả  | Bảo mật  |
| [ ] | Ghi lại lần chạy Lynis; nêu điểm; khắc phục 3 mục                       | Tất cả  | Bảo mật  |

### Sao lưu & khôi phục

| ✓   | Yêu cầu                                                         | Áp dụng | Tiêu chí |
| --- | --------------------------------------------------------------- | ------- | -------- |
| [ ] | Sao lưu tự động: nén, gắn nhãn thời gian, có chính sách lưu giữ | Tất cả  | Sao lưu  |
| [ ] | Lập lịch sao lưu bằng cron hoặc systemd timer; ghi lại lịch     | Tất cả  | Sao lưu  |
| [ ] | Đã luyện quy trình khôi phục; phục hồi dữ liệu thật trực tiếp   | Tất cả  | Sao lưu  |

### Tự động hóa

| ✓   | Yêu cầu                                                                 | Áp dụng | Tiêu chí    |
| --- | ----------------------------------------------------------------------- | ------- | ----------- |
| [ ] | Script health-check kiểm tra trạng thái thật và phản ứng (log/cảnh báo) | Tất cả  | Tự động hóa |
| [ ] | Script dùng `set -euo pipefail` và `trap` trên ERR/EXIT                 | Tất cả  | Tự động hóa |
| [ ] | Mọi script qua shellcheck (sạch, hoặc nêu cảnh báo có lý do)            | Tất cả  | Tự động hóa |
| [ ] | Kênh cảnh báo (mail / msmtp / Telegram) hoạt động khi mô phỏng lỗi      | Tất cả  | Tự động hóa |

### Thành phần 1 — Hạ tầng, Web & Bảo mật (bắt buộc)

| ✓   | Yêu cầu                                                                         | Áp dụng | Tiêu chí |
| --- | ------------------------------------------------------------------------------- | ------- | -------- |
| [ ] | Web server (Nginx/Apache) reverse proxy trước ứng dụng, chuyển tiếp header đúng | Tất cả  | Hệ thống |
| [ ] | ≥2 virtual host trên cùng web server (site app + ít nhất một site nữa)          | Tất cả  | Hệ thống |
| [ ] | Kiến trúc ≥2 VM, có sơ đồ kết nối trong báo cáo                                 | Tất cả  | Hệ thống |

### Thành phần 2 — Ứng dụng & CSDL (bắt buộc)

| ✓   | Yêu cầu                                                                         | Áp dụng | Tiêu chí |
| --- | ------------------------------------------------------------------------------- | ------- | -------- |
| [ ] | Ứng dụng (Flask/FastAPI/Node) có ít nhất một endpoint đọc và một ghi            | Tất cả  | Hệ thống |
| [ ] | CSDL (PostgreSQL hoặc MySQL): role đặc quyền tối thiểu; chỉ nghe localhost (ss) | Tất cả  | Bảo mật  |
| [ ] | App chạy như dịch vụ systemd: Restart=on-failure, EnvironmentFile, journald     | Tất cả  | Hệ thống |
| [ ] | Bí mật trong EnvironmentFile với quyền hạn chế (không hard-code)                | Tất cả  | Bảo mật  |
| [ ] | Kill tiến trình app → systemd tự khởi động lại (thấy trong journalctl)          | Tất cả  | Hệ thống |

### Thành phần 3 — Vận hành & Tự động hóa (bắt buộc)

| ✓   | Yêu cầu                                                                       | Áp dụng | Tiêu chí    |
| --- | ----------------------------------------------------------------------------- | ------- | ----------- |
| [ ] | Menu CLI tương tác điều phối tới từng công cụ; xử lý nhập sai                 | Tất cả  | Tự động hóa |
| [ ] | Deploy có test → reload → xác nhận an toàn và rollback khi lỗi                | Tất cả  | Tự động hóa |
| [ ] | Backup: dump CSDL + web, nén, gắn nhãn, lưu giữ, rsync sang VM thứ hai        | Tất cả  | Sao lưu     |
| [ ] | Restore từ bản sao lưu do bộ công cụ tạo, phục hồi dữ liệu thật               | Tất cả  | Sao lưu     |
| [ ] | Health-check: CPU/disk/RAM/cổng/dịch vụ so ngưỡng, có cảnh báo                | Tất cả  | Tự động hóa |
| [ ] | Xoay vòng log: xoay, nén, giữ N, xóa cũ hơn                                   | Tất cả  | Tự động hóa |
| [ ] | Mỗi công cụ: getopts khi hợp lý, `--help`, ghi log rõ; toàn bộ qua shellcheck | Tất cả  | Tự động hóa |

### Điểm thưởng — tùy chọn (+1 điểm)

| ✓   | Yêu cầu                                                                | Áp dụng | Tiêu chí |
| --- | ---------------------------------------------------------------------- | ------- | -------- |
| [ ] | TLS tự ký phục vụ site/app; chuyển hướng HTTP→HTTPS; mở 443            | Tất cả  | Thưởng   |
| [ ] | Báo cáo giải thích vì sao không dùng được Let's Encrypt trên VM cô lập | Tất cả  | Thưởng   |
