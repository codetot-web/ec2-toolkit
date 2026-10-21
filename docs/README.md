# Tài liệu vận hành ec2-toolkit

Dành cho người vận hành một máy chủ EC2 chạy nhiều site WordPress: Ubuntu 24.04, Apache + PHP-FPM, RDS MySQL, ALB, CloudFlare, sao lưu lên S3.
Mọi tên miền, bucket, endpoint trong tài liệu là ví dụ (`acmeshop`, `example.com`, `your-backups-prod`).

## Đọc theo tình huống

| Tình huống                                               | Đọc                                                    |
| -------------------------------------------------------- | ------------------------------------------------------ |
| Hiểu hệ thống chạy thế nào, những quyết định đã chốt     | [architecture.md](architecture.md)                     |
| Dựng một EC2 mới                                         | [runbook-bootstrap.md](runbook-bootstrap.md)           |
| Thêm một site mới lên EC2                                | [runbook-new-site.md](runbook-new-site.md)             |
| Cài sao lưu S3, khôi phục khi sự cố                      | [runbook-backup-restore.md](runbook-backup-restore.md) |
| Cập nhật PHP, vá khẩn cấp (kể cả CVE-2026-109618/109619) | [runbook-php-update.md](runbook-php-update.md)         |
| Chuyển một site từ BlueHost sang EC2                     | [migration-checklist.md](migration-checklist.md)       |
| Có lỗi, không biết bắt đầu từ đâu                        | [troubleshooting.md](troubleshooting.md)               |
| Tra cứu cờ và hành vi từng script                        | [scripts-reference.md](scripts-reference.md)           |

Tài liệu gốc vẫn giữ nguyên: [../PROJECT-BRIEF.md](../PROJECT-BRIEF.md) (bối cảnh dự án), [../bash-scripts/BACKUP-SETUP.md](../bash-scripts/BACKUP-SETUP.md) (chi tiết S3, IAM, cron),
[../plans/bootstrap-validation.md](../plans/bootstrap-validation.md) (nhật ký kiểm thử bootstrap).

## Điều cần biết trước khi bắt đầu

- Mọi script đều idempotent: chạy lại an toàn. Giữ tính chất này khi sửa script.
- Quy ước tên lệnh: tài liệu gốc gọi `ct-bootstrap`, `ct-create-site`, `ct-backup`, `ct-fix-perm`, `ct-install-php`. Đó là tên khi cài qua `install-tools.sh`.
  Trong repo này các file tên `bootstrap-ec2-wordpress.sh`, `create-site.sh`, `backup-site.sh`. Tài liệu viết cả hai cách khi cần.
- Thư mục site: `/home/ubuntu/webapps/<site>/{public,logs,backups,tmp}`. Chủ sở hữu `ubuntu:www-data`, thư mục `2775`, file `0664`, `.git` thuộc `ubuntu:ubuntu`.
- Chạy wp-cli bằng `sudo -u ubuntu wp ...`. Không bao giờ chạy bằng `root` hoặc `sudo wp`.

## Những thứ chưa có trong repo

Các mục dưới đây được nhắc trong tài liệu gốc nhưng chưa được commit. Đừng giả định nội dung của chúng.

| Thiếu                                       | Hệ quả                                                                                                                             |
| ------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| `install-tools.sh`                          | Chưa có cách cài các lệnh `ct-*` tự động. Chạy trực tiếp bằng `sudo bash <file>.sh`.                                               |
| `install-php-version.sh` (`ct-install-php`) | Thêm PHP phiên bản khác phải làm tay. Xem [runbook-php-update.md](runbook-php-update.md).                                          |
| `fix-permission-site.sh`                    | `create-site.sh` có đoạn dự phòng tự áp quyền. Để sửa quyền sau này, dùng các lệnh trong [troubleshooting.md](troubleshooting.md). |
| `migration-checklist-bluehost-to-aws.md`    | Đã viết lại thành [migration-checklist.md](migration-checklist.md) từ 14 giai đoạn trong PROJECT-BRIEF.                            |
