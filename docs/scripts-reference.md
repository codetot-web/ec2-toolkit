# Tham chiếu script

Mọi script dùng `set -euo pipefail`, mẫu hàm `step_*` và các hàm in `log/ok/warn/err`, và idempotent. Giữ các tính chất này khi chỉnh sửa.

| File                         | Tên lệnh khi cài                 | Chạy bởi        | Tần suất                      |
| ---------------------------- | -------------------------------- | --------------- | ----------------------------- |
| `bootstrap-ec2-wordpress.sh` | `ct-bootstrap`                   | root            | một lần mỗi EC2               |
| `create-site.sh`             | `ct-create-site`                 | root            | một lần mỗi site              |
| `backup-site.sh`             | `ct-backup` (hoặc `backup-site`) | ubuntu qua cron | hằng tuần, tuỳ chọn hằng ngày |

Các lệnh `ct-install-php` và `ct-fix-perm`, và file `install-tools.sh`, được nhắc trong tài liệu gốc nhưng chưa có trong repo.

## bootstrap-ec2-wordpress.sh

```bash
sudo bash bash-scripts/bootstrap-ec2-wordpress.sh
sudo TIMEZONE=UTC bash bash-scripts/bootstrap-ec2-wordpress.sh
```

| Biến           | Mặc định           | Ý nghĩa                                                                             |
| -------------- | ------------------ | ----------------------------------------------------------------------------------- |
| `TIMEZONE`     | `Asia/Ho_Chi_Minh` | múi giờ máy                                                                         |
| `FIX_PERM_URL` | rỗng               | URL script `fix-permission-site.sh` để cài vào `/usr/local/bin/fix-permission-site` |

Các giá trị cố định trong script: PHP `8.3`, thư mục `/home/ubuntu/webapps`, swap 2048 MB, CA RDS tại `/etc/ssl/certs/rds-global-bundle.pem`.
Chi tiết từng bước: [runbook-bootstrap.md](runbook-bootstrap.md).

## create-site.sh

```bash
sudo bash bash-scripts/create-site.sh --site=NAME --domain=DOMAIN [tùy chọn]
```

| Cờ                       | Mặc định              | Ý nghĩa                                                 |
| ------------------------ | --------------------- | ------------------------------------------------------- |
| `--site=NAME`            | bắt buộc              | tên site, `a-z 0-9 _ -`, tối đa 32 ký tự                |
| `--domain=DOMAIN`        | bắt buộc              | tên miền chính                                          |
| `--alias=LIST`           | `www.<domain>`        | các `ServerAlias`, cách nhau bằng dấu phẩy              |
| `--vpc-cidr=CIDR`        | `10.0.0.0/16`         | `RemoteIPInternalProxy` cho ALB                         |
| `--git-repo=URL`         | không                 | clone vào `public/`                                     |
| `--git-branch=NAME`      | `main`                | nhánh clone                                             |
| `--skip-clone`           | tắt                   | bỏ clone dù có `--git-repo`                             |
| `--php-version=VER`      | `8.3`                 | phiên bản PHP-FPM cho pool                              |
| `--memory-limit=VAL`     | `512M`                | `memory_limit`                                          |
| `--upload-max=VAL`       | `64M`                 | `upload_max_filesize` và `post_max_size`                |
| `--max-children=N`       | `20`                  | `pm.max_children`                                       |
| `--rds-host=HOST`        | không                 | endpoint RDS; bỏ qua nghĩa là không tạo DB và wp-config |
| `--rds-master-user=USER` | `admin`               | user quản trị RDS                                       |
| `--rds-master-pass=PASS` | env `RDS_MASTER_PASS` | nên dùng biến môi trường                                |
| `--db-pass=PASS`         | tự sinh               | mật khẩu user ứng dụng                                  |
| `--skip-db`              | tắt                   | bỏ tạo DB dù có `--rds-host`                            |
| `--force`                | tắt                   | ghi đè file cấu hình đã có (không ghi đè database)      |

Chạy bằng `root`. Chi tiết: [runbook-new-site.md](runbook-new-site.md).

## backup-site.sh

```bash
backup-site --site=NAME --bucket=BUCKET [tùy chọn]
```

| Cờ                      | Mặc định                               | Ý nghĩa                                                                      |
| ----------------------- | -------------------------------------- | ---------------------------------------------------------------------------- |
| `--site=NAME`           | bắt buộc                               | phải khớp thư mục site có sẵn                                                |
| `--bucket=BUCKET`       | bắt buộc                               | bucket đã tạo sẵn kèm lifecycle                                              |
| `--prefix=PATH`         | `backups/<site>`                       | tiền tố khoá S3                                                              |
| `--retention-days=N`    | `7`                                    | số ngày giữ `.sql.gz` cục bộ                                                 |
| `--storage-class=CLASS` | `STANDARD_IA`                          | `STANDARD`, `STANDARD_IA`, `ONEZONE_IA`, `INTELLIGENT_TIERING`, `GLACIER_IR` |
| `--region=REGION`       | từ metadata instance hoặc `AWS_REGION` | vùng AWS                                                                     |
| `--skip-db`             | tắt                                    | bỏ sao lưu database                                                          |
| `--skip-uploads`        | tắt                                    | bỏ đồng bộ uploads                                                           |

Cấu trúc trên S3: `<prefix>/db/db-<mốc-thời-gian>.sql.gz` và `<prefix>/uploads/`. Chạy bằng `ubuntu`, xác thực bằng IAM role. Chi tiết: [runbook-backup-restore.md](runbook-backup-restore.md).

## Quy ước khi chỉnh sửa

- Kiểm tra cú pháp: `bash -n bash-scripts/<file>.sh`.
- Thử trên một máy dùng một lần (xem [../plans/bootstrap-validation.md](../plans/bootstrap-validation.md)) trước khi chạy lên máy thật.
- Ghi thay đổi vào [../CHANGELOG.md](../CHANGELOG.md) và tăng `VERSION` nếu đổi hành vi.
- Không đưa mật khẩu, endpoint RDS thật, tên khách hàng thật vào repo (repo công khai).
