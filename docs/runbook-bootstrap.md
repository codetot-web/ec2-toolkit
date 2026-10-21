# Runbook: dựng một EC2 mới

Chạy một lần cho mỗi máy. Script `bootstrap-ec2-wordpress.sh` chạy bằng `root` trên Ubuntu 24.04 sạch, và chạy lại an toàn.

## Trước khi chạy

- EC2 Ubuntu 24.04, user `ubuntu` có `sudo`.
- Security group: cổng 22 chỉ từ IP máy trung gian; cổng 80 từ ALB.
- RDS và IAM role cho S3 có thể chuẩn bị song song (xem [runbook-backup-restore.md](runbook-backup-restore.md)).
- Máy có ít nhất vài GB trống cho cập nhật gói.

## Chạy

```bash
sudo bash bash-scripts/bootstrap-ec2-wordpress.sh
```

Biến môi trường tuỳ chọn:

```bash
sudo TIMEZONE=UTC bash bash-scripts/bootstrap-ec2-wordpress.sh
```

`TIMEZONE` mặc định là `Asia/Ho_Chi_Minh`. `FIX_PERM_URL` cài thêm lệnh `fix-permission-site` từ URL bạn chỉ định (chỉ khi có script đó, xem [README.md](README.md)).

## Script làm gì (theo thứ tự)

| Bước              | Kết quả                                                                                                                                                                          |
| ----------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Cập nhật hệ thống | `apt full-upgrade`                                                                                                                                                               |
| Múi giờ           | đặt theo `TIMEZONE`                                                                                                                                                              |
| Gói               | Apache2, PHP 8.3-FPM và các extension WordPress (mysql, curl, gd, mbstring, xml, zip, intl, bcmath, soap, opcache, imagick, redis), Redis, mysql-client, git, acl, ufw, fail2ban |
| Apache            | tắt mod_php và prefork; bật `proxy_fcgi`, `rewrite`, `headers`, `expires`, `deflate`, `ssl`, `mpm_event`, `remoteip`; ẩn phiên bản (`ServerTokens Prod`)                         |
| WP-CLI            | cài ở `/usr/local/bin/wp`, bỏ qua nếu đã có                                                                                                                                      |
| CA của RDS        | `/etc/ssl/certs/rds-global-bundle.pem`, bỏ qua nếu đã có và hợp lệ                                                                                                               |
| User              | `ubuntu` và `www-data` là thành viên nhóm của nhau; `/home/ubuntu` là `755`; `umask 002` cho ubuntu; `git safe.directory` cho cây webapps                                        |
| Thư mục           | `/home/ubuntu/webapps` thuộc `ubuntu:www-data`, mode `2755`                                                                                                                      |
| Swap              | 2 GB nếu chưa có swap nào, `vm.swappiness=10`                                                                                                                                    |
| Tường lửa         | UFW mở 22, 80, 443, chặn còn lại                                                                                                                                                 |
| fail2ban          | bật jail sshd                                                                                                                                                                    |
| Redis             | bật                                                                                                                                                                              |

## Kiểm tra sau khi chạy

```bash
php -v
apache2ctl -M | grep -E 'proxy_fcgi|mpm_event|remoteip'
systemctl is-active apache2 php8.3-fpm redis-server fail2ban
sudo ufw status
sudo -u ubuntu wp --info | head -5
stat -c '%U:%G %a' /home/ubuntu/webapps
```

Kết quả mong đợi: PHP 8.3, ba module Apache có mặt, bốn dịch vụ `active`, UFW `active` với 22/80/443, thư mục webapps là `ubuntu:www-data 2755`.

Nếu cuối bản tóm tắt có cảnh báo `reboot-required`, khởi động lại máy trong cửa sổ bảo trì rồi kiểm tra lại các dịch vụ.

## Việc cần làm tiếp

1. Thêm site: [runbook-new-site.md](runbook-new-site.md).
2. Thiết lập sao lưu: [runbook-backup-restore.md](runbook-backup-restore.md).
3. Kiểm tra DB từ EC2: cần mạng tới RDS và CA bundle ở đúng chỗ.

## Sự cố thường gặp

| Triệu chứng                   | Cách xử lý                                                                                                            |
| ----------------------------- | --------------------------------------------------------------------------------------------------------------------- |
| Mất SSH khi bật UFW           | Không xảy ra nếu chạy đúng script (cổng 22 được mở trước khi `default deny`). Nếu tự chỉnh UFW, giữ nguyên thứ tự đó. |
| `apt` khoá                    | Đợi `unattended-upgrades` kết thúc: `ps aux                                                                           | grep -E 'apt | dpkg'` |
| Hai swap                      | Script chỉ tạo `/swapfile` khi chưa có swap nào; kiểm tra bằng `swapon --show`                                        |
| Chạy trên host không phải EC2 | Cần có user `ubuntu`. Xem [../plans/bootstrap-validation.md](../plans/bootstrap-validation.md)                        |
