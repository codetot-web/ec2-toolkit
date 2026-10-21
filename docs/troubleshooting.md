# Xử lý sự cố

Bắt đầu từ triệu chứng. Với sự cố sao lưu, xem thêm [runbook-backup-restore.md](runbook-backup-restore.md).

## Site trả về lỗi

| Triệu chứng                                | Kiểm tra                                                                                                       | Hướng xử lý                                                                                   |
| ------------------------------------------ | -------------------------------------------------------------------------------------------------------------- | --------------------------------------------------------------------------------------------- |
| 502 từ ALB                                 | `systemctl status apache2`; `apachectl configtest`                                                             | Sửa lỗi cấu hình, `systemctl reload apache2`                                                  |
| 503 hoặc trang PHP không tải               | `ls -l /run/php/<site>.sock`; `systemctl status php8.3-fpm`                                                    | Pool bị lỗi: `php-fpm8.3 -t`, xem `logs/php-error.log`                                        |
| Trang trắng, 500                           | `tail -50 /home/ubuntu/webapps/<site>/logs/error.log` và `php-error.log`                                       | Plugin hoặc theme lỗi: tắt plugin bằng `sudo -u ubuntu wp plugin deactivate <tên> --path=...` |
| "Error establishing a database connection" | `sudo -u ubuntu wp --path=... db check`                                                                        | Kiểm tra security group RDS, mật khẩu trong `.credentials`, CA bundle tồn tại                 |
| Vòng lặp chuyển hướng HTTPS                | Vhost có `SetEnvIf X-Forwarded-Proto "https" HTTPS=on` không; wp-config có đoạn nhận `X-Forwarded-Proto` không | Tạo lại wp-config bằng `create-site.sh --force`                                               |
| Log chỉ thấy IP của ALB                    | `RemoteIPInternalProxy` trong vhost có khớp CIDR của VPC không                                                 | Đặt `--vpc-cidr` đúng khi tạo site                                                            |

## Database

```bash
sudo -u ubuntu wp --path=/home/ubuntu/webapps/<site>/public db check
cat /home/ubuntu/webapps/<site>/.credentials
ls -l /etc/ssl/certs/rds-global-bundle.pem
```

- Người dùng database có `REQUIRE SSL`; kết nối không dùng SSL sẽ bị từ chối. wp-config phải có `MYSQL_CLIENT_FLAGS` với `MYSQLI_CLIENT_SSL`.
- `create-site.sh` chạy lại an toàn: nó đồng bộ mật khẩu database với `.credentials` bằng `ALTER USER`. Truyền `RDS_MASTER_PASS` qua môi trường.
- Chạy wp-cli luôn bằng `sudo -u ubuntu`. Chạy bằng `root` tạo file thuộc root và làm hỏng quyền.

## Quyền file

Triệu chứng: `git pull` báo permission denied, WordPress không ghi được uploads, plugin báo không tạo được thư mục.

Chuẩn: thư mục site `ubuntu:www-data` `2775`, file `0664`, `.git` thuộc `ubuntu:ubuntu`, `wp-config.php` `640`, `.credentials` `600`.

`fix-permission-site.sh` chưa có trong repo. Dùng các lệnh sau (sửa `SITE_ROOT`):

```bash
SITE_ROOT=/home/ubuntu/webapps/acmeshop
PUBLIC_DIR=$SITE_ROOT/public
sudo chown -R ubuntu:www-data "$SITE_ROOT"
sudo find "$SITE_ROOT" -path "*/.git" -prune -o -type d -exec chmod 2775 {} +
sudo find "$SITE_ROOT" -path "*/.git" -prune -o -type f -exec chmod 0664 {} +
[ -d "$PUBLIC_DIR/.git" ] && sudo chown -R ubuntu:ubuntu "$PUBLIC_DIR/.git" \
  && sudo find "$PUBLIC_DIR/.git" -type d -exec chmod 700 {} + \
  && sudo find "$PUBLIC_DIR/.git" -type f -exec chmod 600 {} +
[ -f "$PUBLIC_DIR/wp-config.php" ] && sudo chmod 640 "$PUBLIC_DIR/wp-config.php"
[ -f "$SITE_ROOT/.credentials" ] && sudo chmod 600 "$SITE_ROOT/.credentials"
```

Các lệnh này giống phần dự phòng trong `create-site.sh`. Không dùng `chown -R www-data:www-data` và không dùng `chmod 777`.

## Git

| Triệu chứng                                | Nguyên nhân                  | Xử lý                                                                                     |
| ------------------------------------------ | ---------------------------- | ----------------------------------------------------------------------------------------- |
| "dubious ownership"                        | `safe.directory` chưa có     | `sudo -u ubuntu git config --global --add safe.directory '/home/ubuntu/webapps/*/public'` |
| `git pull` bị từ chối quyền                | file hoặc `.git` sai chủ     | Áp lại quyền như trên                                                                     |
| Plugin cài từ wp-admin biến mất khi deploy | Cài bằng wp-admin bỏ qua git | Đưa plugin vào git hoặc bật `DISALLOW_FILE_MODS`                                          |

## Mạng và tường lửa

- Mất SSH sau khi đổi UFW: đừng reset UFW qua phiên SSH. Thứ tự đúng là `ufw allow 22` rồi mới `ufw default deny`.
- Không SSH thẳng từ internet: đi qua máy trung gian (RDP → Bitvise) theo ràng buộc đã nêu trong [architecture.md](architecture.md).

## WP-Cron không chạy

`create-site.sh` bật `DISABLE_WP_CRON`. Kiểm tra cron của `ubuntu`: `crontab -l`. Nếu thiếu, thêm dòng ở [runbook-new-site.md](runbook-new-site.md).

## Thu thập thông tin khi cần hỗ trợ

```bash
lsb_release -ds
php -v | head -1
systemctl is-active apache2 php8.3-fpm redis-server
tail -50 /home/ubuntu/webapps/<site>/logs/error.log
tail -50 /home/ubuntu/webapps/<site>/logs/php-error.log
cat /home/ubuntu/webapps/<site>/.credentials | grep -v DB_PASS
cat /path/to/repo/VERSION
```

Không dán mật khẩu, nội dung `.credentials`, hoặc endpoint RDS vào ticket công khai.
