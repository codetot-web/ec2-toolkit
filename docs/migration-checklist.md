# Checklist chuyển site từ BlueHost sang EC2

Ví dụ trong tài liệu dùng site `acmeshop` (`acmeshop.example.com`). Mỗi giai đoạn có đầu ra rõ ràng; chỉ sang giai đoạn tiếp theo khi giai đoạn trước đã xong.
Danh sách 14 giai đoạn lấy từ [../PROJECT-BRIEF.md](../PROJECT-BRIEF.md).

## 1. Kiểm kê BlueHost

- [ ] Danh sách plugin, theme, phiên bản PHP đang chạy
- [ ] Dung lượng database và uploads
- [ ] Bản ghi DNS hiện có, nhà cung cấp DNS
- [ ] Có dùng hộp thư BlueHost không. Nếu có, di chuyển email là việc riêng và phải xong trước khi đổi DNS
- [ ] Hạ TTL của bản ghi cần đổi xuống 300 giây, ít nhất 48 giờ trước ngày chuyển

## 2. Hạ tầng AWS

- [ ] VPC, subnet, security group (`sg-ec2`, `sg-rds`, ALB)
- [ ] EC2 Ubuntu 24.04; cổng 22 chỉ mở cho IP máy trung gian
- [ ] RDS MySQL 8.0 trong mạng riêng, `require_secure_transport=ON`
- [ ] ALB với chứng chỉ ACM (do đội khác cung cấp; theo dõi như một phụ thuộc)

## 3. Cài nền cho EC2

- [ ] Chạy `bootstrap-ec2-wordpress.sh` ([runbook-bootstrap.md](runbook-bootstrap.md))

## 4. Chiến lược quyền file

- [ ] `ubuntu` và `www-data` là thành viên nhóm của nhau (bootstrap đã làm)
- [ ] Quy ước `ubuntu:www-data`, thư mục `2775`, file `0664`; `.git` thuộc `ubuntu:ubuntu`

## 5. Cấu trúc thư mục nhiều site

- [ ] Chạy `create-site.sh` cho site ([runbook-new-site.md](runbook-new-site.md))

## 6. Repo git

- [ ] `.gitignore` đầy đủ cho cả dự án WordPress (loại uploads, cache, bí mật)
- [ ] `wp-config.sample.php` được commit; `wp-config.php` thật chỉ có trên máy chủ
- [ ] Deploy key riêng cho từng repo (alias SSH `github.com-<site>`)
- [ ] Quyết định chính sách cài plugin trong wp-admin: `DISALLOW_FILE_MODS=true` (khoá, do dev kiểm soát) hoặc đưa ngược về git

## 7. Xuất từ BlueHost

- [ ] `wp db export` trên BlueHost
- [ ] Nén uploads thành một file tar
- [ ] Chuyển qua Bitvise SFTP vào `/home/ubuntu/webapps/acmeshop/backups/`

## 8. Nhập vào EC2

```bash
cd /home/ubuntu/webapps/acmeshop/public
gunzip -c /home/ubuntu/webapps/acmeshop/backups/<file>.sql.gz | sudo -u ubuntu wp db import -
sudo -u ubuntu wp search-replace 'http://acmeshop.example.com' 'https://acmeshop.example.com' --all-tables --skip-columns=guid
cd wp-content && rm -rf uploads && tar -xzf ../../backups/<uploads>.tar.gz
```

- [ ] Áp lại quyền ([troubleshooting.md](troubleshooting.md), mục quyền file)
- [ ] Đặt cron hệ thống cho WP-Cron (`DISABLE_WP_CRON` đang bật, xem [runbook-new-site.md](runbook-new-site.md))

## 9. Truy cập chéo staging

- [ ] Khoá SSH từ EC2 tới máy staging
- [ ] Quy tắc rsync cho nội dung (uploads, bản dump DB)
- [ ] Mã nguồn chỉ đi qua git, không rsync mã

## 10. Cấu hình CloudFlare

- [ ] Bản ghi DNS, chế độ SSL Full (Strict)
- [ ] WAF và quy tắc cache
- [ ] Security group của ALB chỉ cho dải IP của CloudFlare, để không đi vòng qua CloudFlare

## 11. Phối hợp SSL

- [ ] Chứng chỉ ACM và chứng chỉ CloudFlare do đội khác xử lý; xác nhận trước ngày chuyển

## 12. Chuyển DNS

- [ ] Kiểm tra bằng sửa file `hosts` trước
- [ ] Đồng bộ delta database lần cuối, đóng băng thay đổi nội dung trên BlueHost
- [ ] Đổi DNS ở CloudFlare
- [ ] Smoke test: trang chủ, đăng nhập admin, một bài viết, form, giỏ hàng nếu có
- [ ] Bật HSTS sau 24 giờ nếu mọi thứ ổn

## 13. Kiểm tra sau chuyển

- [ ] SSL hoạt động, không có mixed content
- [ ] Log hiển thị IP thật của khách, không phải IP của ALB hay CloudFlare
- [ ] DB dùng SSL: `SHOW STATUS LIKE 'Ssl_cipher'` trả về giá trị không rỗng
- [ ] `.git` không truy cập được từ web
- [ ] Cron WP-Cron chạy (`wp cron event list`)

## 14. Sao lưu, giám sát, củng cố

- [ ] Cron sao lưu ([runbook-backup-restore.md](runbook-backup-restore.md)) và đã thử khôi phục một lần
- [ ] Cảnh báo CloudWatch
- [ ] NinjaFirewall
- [ ] `DISALLOW_FILE_EDIT`; bật 2FA cho admin; xoá tài khoản `admin` cũ từ thời BlueHost

## Quay lại (rollback)

Giữ site BlueHost chạy nguyên trạng ít nhất 7 ngày sau khi đổi DNS. Nếu cần quay lại trong thời gian TTL còn thấp, đổi DNS về BlueHost và đồng bộ ngược các thay đổi nội dung đã phát sinh trên EC2 (đơn hàng, bình luận).
