# Kiến trúc và quyết định đã chốt

## Luồng yêu cầu

```
Người dùng → CloudFlare (proxy, WAF, cache biên)
          → ALB (chỉ HTTPS, chứng chỉ ACM)
          → EC2 (Ubuntu 24.04, Apache 2.4 mpm_event + PHP-FPM, nhiều site)
          → RDS MySQL 8.0 (mạng riêng, bắt buộc SSL)

EC2 đi ra:
  → S3 (sao lưu, xác thực bằng IAM role của instance)
  → GitHub (deploy key riêng cho từng repo)
```

ALB kết thúc TLS. Apache chỉ nghe cổng 80 và tin header `X-Forwarded-Proto`. Không có certbot trên máy.

## Cấu trúc một site

```
/home/ubuntu/webapps/<site>/
├── public/        DocumentRoot của Apache; .git nằm ở đây (chỉ ubuntu đọc ghi)
├── logs/          access.log, error.log, php-error.log
├── backups/       bản dump DB cục bộ (giữ 7 ngày)
├── tmp/           session và thư mục tạm upload của PHP
└── .credentials   chmod 600, ubuntu:ubuntu, thông tin database
```

| Thành phần        | Vị trí                                        |
| ----------------- | --------------------------------------------- |
| Apache vhost      | `/etc/apache2/sites-available/<site>.conf`    |
| PHP-FPM pool      | `/etc/php/<phiên bản>/fpm/pool.d/<site>.conf` |
| Socket PHP-FPM    | `/run/php/<site>.sock`                        |
| CA bundle của RDS | `/etc/ssl/certs/rds-global-bundle.pem`        |

Mỗi site có một pool PHP-FPM riêng (`pm = ondemand`, `max_children` mặc định 20, `memory_limit` 512M) và một vhost riêng, nên site này lỗi không kéo site khác.

## Quyết định đã chốt, không tranh luận lại

| Quyết định                                                      | Lý do                                                                                                       |
| --------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------- |
| Apache + PHP-FPM, không dùng Nginx                              | Đội quen Apache; pool riêng đã đủ cách ly                                                                   |
| File site thuộc `ubuntu:www-data`, `.git` thuộc `ubuntu:ubuntu` | Cho phép `git pull` bằng ubuntu mà web vẫn ghi được; plugin chạy bằng `www-data` không phá được lịch sử git |
| Tên database == tên user == tên site                            | Một tên cho một dự án, script ép buộc                                                                       |
| RDS bắt buộc SSL                                                | `require_secure_transport=ON`, user tạo với `REQUIRE SSL`                                                   |
| ALB kết thúc TLS                                                | Chứng chỉ ACM do đội khác quản lý                                                                           |
| CloudFlare đứng trước ALB                                       | IP thật lấy từ `HTTP_CF_CONNECTING_IP`, gán vào `REMOTE_ADDR` trong wp-config                               |
| Git là kênh phân phối mã                                        | Cài plugin trong wp-admin bỏ qua git, nên cần `DISALLOW_FILE_MODS` hoặc quy trình đưa ngược về git          |
| Mọi thao tác DB qua WP-CLI                                      | `wp db export` và `wp db import`, không gọi `mysqldump` trực tiếp                                           |
| Sao lưu bằng `aws s3 sync`                                      | rsync không nói chuyện được với S3                                                                          |
| PHP nhiều phiên bản qua Ondrej PPA                              | Ubuntu 24.04 mặc định 8.3; phiên bản cũ cài song song                                                       |
| AWS CLI v2 từ URL của AWS                                       | Gói `awscli` của apt là v1, bị cấm                                                                          |

## Những việc làm hỏng mô hình này

- `chown -R www-data:www-data` lên file site: khoá `ubuntu` khỏi `git pull`.
- `chmod 777`: không cần, vì setgid cộng quyền nhóm đã đủ cho cả hai user.
- Chạy wp-cli bằng `root` hoặc `sudo wp`.
- Để CA bundle của RDS trong `/home/ubuntu/`: `www-data` có thể không đọc được.
- Rsync mã nguồn giữa EC2 và staging. Mã đi qua git; rsync chỉ dành cho uploads và bản dump DB.
- `ufw --force reset` giữa phiên SSH: bootstrap xếp `ufw allow 22` trước `ufw default deny` có chủ đích.

## Ràng buộc

- Truy cập EC2: Windows RDP → Bitvise SSH (khoá `.pem`) → EC2. Không SSH thẳng từ internet; security group chỉ mở cổng 22 cho IP đi ra của máy trung gian.
- Chứng chỉ SSL và CloudFlare do đội khác xử lý, coi là phụ thuộc.
- Email: nếu còn dùng hộp thư BlueHost, di chuyển email là một việc riêng và phải xong trước khi đổi DNS.

## wp-config do `create-site.sh` sinh ra

Thêm các dòng: nhận diện HTTPS qua `X-Forwarded-Proto`, IP thật từ CloudFlare, cờ SSL cho kết nối RDS (`MYSQL_CLIENT_FLAGS`), `WP_HOME` và `WP_SITEURL`,
`FORCE_SSL_ADMIN`, `DISABLE_WP_CRON` (bật), `WP_AUTO_UPDATE_CORE` là `minor`, `DISALLOW_FILE_EDIT`.
`DISABLE_WP_CRON` bật nghĩa là WP-Cron không tự chạy theo lượt truy cập; phải đặt cron hệ thống gọi `wp cron event run`. Xem [runbook-new-site.md](runbook-new-site.md).
