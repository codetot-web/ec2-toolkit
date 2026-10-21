# Runbook: thêm một site mới

`create-site.sh` chạy bằng `root`, một lần cho mỗi site, sau khi `bootstrap-ec2-wordpress.sh` đã chạy xong trên máy.

## Chuẩn bị

| Cần có                            | Ghi chú                                                                                                  |
| --------------------------------- | -------------------------------------------------------------------------------------------------------- |
| Tên site                          | Chỉ `a-z 0-9 _ -`, tối đa 32 ký tự. Dùng làm tên thư mục, vhost, pool PHP-FPM, database và user database |
| Tên miền                          | Ví dụ `acmeshop.example.com`; mặc định thêm alias `www.<tên miền>`                                       |
| Repo git (nếu có)                 | Dùng alias SSH `github.com-<site>` đã cấu hình deploy key                                                |
| Endpoint RDS và mật khẩu quản trị | Mật khẩu truyền qua biến môi trường để không nằm trong lịch sử shell                                     |

## Chạy

Đầy đủ (repo, RDS, vhost, pool, wp-config):

```bash
sudo RDS_MASTER_PASS='masterPass' bash bash-scripts/create-site.sh \
    --site=acmeshop \
    --domain=acmeshop.example.com \
    --git-repo=git@github.com-acmeshop:your-org-clients/acmeshop.git \
    --rds-host=mydb.abc123.ap-southeast-1.rds.amazonaws.com
```

Site rỗng (chỉ khung thư mục, vhost, pool):

```bash
sudo bash bash-scripts/create-site.sh --site=staging --domain=staging.example.com
```

Site nặng hơn:

```bash
sudo RDS_MASTER_PASS='masterPass' bash bash-scripts/create-site.sh \
    --site=bigshop --domain=bigshop.com \
    --memory-limit=1024M --upload-max=128M --max-children=40 \
    --rds-host=mydb.abc123.ap-southeast-1.rds.amazonaws.com
```

Các cờ khác: `--alias`, `--vpc-cidr` (mặc định `10.0.0.0/16`, dùng cho `RemoteIPInternalProxy`), `--git-branch`, `--php-version` (mặc định `8.3`),
`--db-pass`, `--skip-db`, `--skip-clone`, `--force`. Xem đầy đủ: `bash create-site.sh --help`.

`--force` ghi đè file cấu hình và thư mục public đã có, nhưng không ghi đè database đã có.

## Script làm gì (theo thứ tự)

1. Tạo `public`, `logs`, `backups`, `tmp`.
2. Clone repo vào `public/` nếu có `--git-repo` (từ chối nếu `public/` không rỗng và không có `--force`).
3. Tạo database và user trùng tên site trên RDS, `REQUIRE SSL`, rồi kiểm tra user đăng nhập được. Chỉ chạy khi có `--rds-host`.
4. Ghi `.credentials` (mode 600, `ubuntu:ubuntu`) chứa mật khẩu database tự sinh.
5. Tạo `wp-config.php` bằng WP-CLI, nếu tìm thấy lõi WordPress trong `public/`. Nếu chưa có lõi, script cảnh báo và hướng dẫn tải lõi rồi chạy lại với `--force`.
6. Ghi Apache vhost và bật site.
7. Ghi pool PHP-FPM (`pm = ondemand`, `max_requests = 500`, socket `/run/php/<site>.sock`).
8. Kiểm tra cấu hình (`apachectl configtest`), reload PHP-FPM và Apache.
9. Áp quyền `ubuntu:www-data` thư mục `2775`, file `0664`, `.git` về `ubuntu:ubuntu`, `wp-config.php` mode `640`.

## Sau khi chạy

```bash
cat /home/ubuntu/webapps/acmeshop/.credentials
ls -l /run/php/acmeshop.sock
sudo apache2ctl -S | grep acmeshop
sudo -u ubuntu wp --path=/home/ubuntu/webapps/acmeshop/public core is-installed && echo OK
curl -I -H 'Host: acmeshop.example.com' -H 'X-Forwarded-Proto: https' http://127.0.0.1/
```

Hãy cất mật khẩu trong `.credentials` vào nơi quản lý bí mật của đội. Không commit file này.

## Cron WP-Cron

`create-site.sh` bật `DISABLE_WP_CRON`. WordPress sẽ không tự chạy tác vụ định kỳ nữa, nên cần cron hệ thống (crontab của `ubuntu`):

```cron
*/5 * * * * cd /home/ubuntu/webapps/acmeshop/public && /usr/local/bin/wp cron event run --due-now >/dev/null 2>&1
```

Nếu bỏ qua bước này, bài hẹn giờ, email đơn hàng, và các plugin dựa vào cron sẽ ngừng chạy.

## Nhập dữ liệu có sẵn

Xem [migration-checklist.md](migration-checklist.md), giai đoạn 7 và 8. Tóm tắt:

```bash
cd /home/ubuntu/webapps/acmeshop/public
gunzip -c /home/ubuntu/webapps/acmeshop/backups/<file>.sql.gz | sudo -u ubuntu wp db import -
sudo -u ubuntu wp search-replace 'http://acmeshop.example.com' 'https://acmeshop.example.com' --all-tables --skip-columns=guid
```

## Bật sao lưu cho site mới

Thêm một dòng cron của `ubuntu`, so le 30 phút giữa các site: xem [runbook-backup-restore.md](runbook-backup-restore.md).

## Gỡ một site

Chưa có script. Thứ tự an toàn: sao lưu lần cuối lên S3, `a2dissite <site>`, xoá pool trong `/etc/php/<phiên bản>/fpm/pool.d/`, reload Apache và PHP-FPM,
xoá database và user trên RDS, rồi mới xoá thư mục site. Làm theo từng bước và xác nhận trước khi xoá.
