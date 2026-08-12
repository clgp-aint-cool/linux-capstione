# Checklist chia theo từng người

Dựa theo bảng kiểm ở Mục 12 tài liệu, đánh dấu ✓ nghĩa là người đó là **chủ trì**; mục có ghi "(phối hợp: …)" là mục cần góp sức từ người khác.

---

## Phú — Hạ tầng & mạng

**Hệ thống & lưu trữ**

- [ ] Cài Ubuntu 22.04 hoặc CentOS Stream 9 bản sạch trên cả 2 VM; đặt hostname có ý nghĩa
- [ ] Dùng tài khoản quản trị non-root (sudo) cho công việc hằng ngày
- [ ] Vùng dữ liệu/sao lưu riêng mount qua `/etc/fstab`, tồn tại sau reboot
- [ ] Kiến trúc từ 2 VM trở lên — dựng VM, mạng Host-Only/Internal, vẽ sơ đồ kết nối _(phối hợp: Phúc — cổng dịch vụ; Khánh — VM2 nhận backup)_

---

## Phúc — Reverse proxy & bảo mật web

**Gia cố bảo mật (toàn hệ thống)**

- [ ] Tường lửa (UFW hoặc firewalld) mặc định chặn vào; chỉ mở cổng cần thiết
- [ ] Gia cố SSH: tắt root, xác thực khóa, đổi cổng hoặc AllowUsers
- [ ] fail2ban bảo vệ SSH; ban được kích hoạt trực tiếp
- [ ] auditd theo dõi `/etc/passwd` và `/etc/shadow`; dấu vết đọc được
- [ ] Ghi lại lần chạy Lynis; nêu điểm; khắc phục 3 mục

**Thành phần 1 — Hạ tầng, Web & Bảo mật**

- [ ] Web server (Nginx/Apache) reverse proxy trước ứng dụng, chuyển tiếp header đúng _(phối hợp: Duy — cổng app cục bộ)_
- [ ] ≥2 virtual host trên cùng web server (site app + ít nhất một site nữa)
- [ ] Kiến trúc ≥2 VM, có sơ đồ kết nối trong báo cáo _(phối hợp: Phú)_

**Điểm thưởng (tùy chọn)**

- [ ] TLS tự ký phục vụ site/app; chuyển hướng HTTP→HTTPS; mở 443
- [ ] Báo cáo giải thích vì sao không dùng được Let's Encrypt trên VM cô lập

---

## Duy — Ứng dụng & systemd

**Thành phần 2 — Ứng dụng & CSDL**

- [ ] Ứng dụng (Flask/FastAPI/Node) có ít nhất một endpoint đọc và một ghi _(phối hợp: Khánh — schema/connection)_
- [ ] App chạy như dịch vụ systemd: `Restart=on-failure`, `EnvironmentFile`, journald
- [ ] Bí mật trong `EnvironmentFile` với quyền hạn chế (không hard-code)
- [ ] Kill tiến trình app → systemd tự khởi động lại (thấy trong `journalctl`)

---

## Khánh — CSDL & sao lưu

**Thành phần 2 — Ứng dụng & CSDL**

- [ ] CSDL (PostgreSQL hoặc MySQL): role đặc quyền tối thiểu; chỉ nghe localhost (kiểm chứng bằng `ss`)

**Sao lưu & khôi phục (toàn hệ thống)**

- [ ] Sao lưu tự động: nén, gắn nhãn thời gian, có chính sách lưu giữ — viết logic dump CSDL _(phối hợp: Long — tích hợp vào script backup chung + nội dung web)_
- [ ] Lập lịch sao lưu bằng cron hoặc systemd timer; ghi lại lịch _(phối hợp: Long)_
- [ ] Đã luyện quy trình khôi phục; phục hồi dữ liệu thật trực tiếp — chuẩn bị kịch bản demo restore CSDL

**Thành phần 3 — góp phần**

- [ ] Backup: cung cấp lệnh `mysqldump`/`pg_dump` chuẩn cho Long tích hợp vào script backup + rsync sang VM2
- [ ] Restore: xác nhận logic khôi phục CSDL đúng khi Long gọi từ bộ công cụ

---

## Long — Bộ công cụ tự động hóa & cảnh báo

**Tự động hóa (toàn hệ thống)**

- [ ] Script health-check kiểm tra trạng thái thật và phản ứng (log/cảnh báo)
- [ ] Script dùng `set -euo pipefail` và `trap` trên ERR/EXIT
- [ ] Mọi script qua shellcheck (sạch, hoặc nêu cảnh báo có lý do)
- [ ] Kênh cảnh báo (mail / msmtp / Telegram) hoạt động khi mô phỏng lỗi

**Thành phần 3 — Vận hành & Tự động hóa**

- [ ] Menu CLI tương tác điều phối tới từng công cụ; xử lý nhập sai
- [ ] Deploy có test → reload → xác nhận an toàn và rollback khi lỗi _(phối hợp: Phúc — reload web server; Duy — restart app service)_
- [ ] Backup: dump CSDL + web, nén, gắn nhãn, lưu giữ, rsync sang VM thứ hai _(phối hợp: Khánh)_
- [ ] Restore từ bản sao lưu do bộ công cụ tạo, phục hồi dữ liệu thật _(phối hợp: Khánh)_
- [ ] Health-check: CPU/disk/RAM/cổng/dịch vụ so ngưỡng, có cảnh báo
- [ ] Xoay vòng log: xoay, nén, giữ N, xóa cũ hơn
- [ ] Mỗi công cụ: getopts khi hợp lý, `--help`, ghi log rõ; toàn bộ qua shellcheck

---
