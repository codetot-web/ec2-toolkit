# Runbook: cập nhật và vá PHP

Repo chưa có script cho việc này (`install-php-version.sh` được nhắc trong tài liệu gốc nhưng chưa tồn tại). Đây là quy trình thủ công. Mọi lệnh chạy bằng `root` hoặc `sudo`.

## Bố cục PHP trên máy này

| Thành phần     | Vị trí                                                                                     |
| -------------- | ------------------------------------------------------------------------------------------ |
| Gói mặc định   | PHP 8.3 từ Ubuntu 24.04 (`php8.3-fpm`)                                                     |
| Phiên bản khác | Cài song song từ `ppa:ondrej/php`                                                          |
| Dịch vụ        | `php<phiên bản>-fpm`, mỗi phiên bản một dịch vụ                                            |
| Pool từng site | `/etc/php/<phiên bản>/fpm/pool.d/<site>.conf`, socket `/run/php/<site>.sock`               |
| Vhost          | `/etc/apache2/sites-available/<site>.conf` có `SetHandler "proxy:unix:/run/php/<site>.sock | fcgi://localhost"` |

Một site đổi phiên bản PHP bằng cách chuyển file pool của nó sang thư mục `pool.d` của phiên bản mới; socket giữ nguyên nên vhost không phải sửa.

## Bước 1: kiểm kê

```bash
dpkg -l 'php*-fpm' | awk '/^ii/{print $2, $3}'
systemctl list-units --type=service --state=running | grep -E 'php[0-9.]+-fpm'
ls /etc/php/*/fpm/pool.d/
for p in /etc/php/*/fpm/pool.d/*.conf; do echo "$p"; done
```

Đối chiếu pool với site để biết site nào chạy phiên bản nào. Pool tên `www.conf` là pool mặc định, không phải site.

Tra tình trạng hỗ trợ ở [php.net/supported-versions.php](https://www.php.net/supported-versions.php). Tại 21/10/2026: 8.4 và 8.5 hỗ trợ đầy đủ; 8.2 và 8.3 chỉ vá bảo mật; 8.1 và 7.4 đã hết hỗ trợ.

## Bước 2: thêm một phiên bản PHP song song (khi cần nâng nhánh)

```bash
add-apt-repository -y ppa:ondrej/php
apt update
apt install -y php8.4-fpm php8.4-mysql php8.4-curl php8.4-gd php8.4-mbstring php8.4-xml \
    php8.4-zip php8.4-intl php8.4-bcmath php8.4-soap php8.4-opcache php8.4-redis php8.4-imagick
systemctl enable --now php8.4-fpm
```

Danh sách extension lấy theo `bootstrap-ec2-wordpress.sh` để site chạy giống như trên phiên bản mặc định. Nếu một extension không có bản cho phiên bản mới, báo trước khi chuyển site.

## Bước 3: chuyển một site sang phiên bản khác

1. Chọn site ít quan trọng và có bản sao lưu mới ([runbook-backup-restore.md](runbook-backup-restore.md)).
2. Sao chép pool sang phiên bản mới (thay `8.3` và `8.4`, và `acmeshop` cho đúng):
   ```bash
   cp /etc/php/8.3/fpm/pool.d/acmeshop.conf /etc/php/8.4/fpm/pool.d/acmeshop.conf
   mv /etc/php/8.3/fpm/pool.d/acmeshop.conf /root/acmeshop.pool.8.3.bak
   ```
3. Kiểm tra cấu hình và nạp lại:
   ```bash
   php-fpm8.4 -t
   systemctl reload php8.3-fpm php8.4-fpm
   apache2ctl configtest && systemctl reload apache2
   ```
4. Kiểm tra site: trang chủ, đăng nhập admin, một form, `tail /home/ubuntu/webapps/acmeshop/logs/php-error.log`.
5. Nếu lỗi, hoàn tác: chuyển file pool về và nạp lại hai dịch vụ.

## Bước 4: vá một phiên bản đang dùng

```bash
apt update
apt list --upgradable 2>/dev/null | grep -i php
apt install --only-upgrade php8.3-fpm php8.3-cli php8.3-common php8.3-mysql php8.3-curl php8.3-gd php8.3-mbstring php8.3-xml php8.3-zip php8.3-intl php8.3-bcmath php8.3-soap php8.3-opcache
php-fpm8.3 -t
systemctl reload php8.3-fpm
php-fpm8.3 -v
```

Dùng `reload` thay vì `restart` để giữ kết nối đang có. Sau khi nâng cấp gói, xác nhận tiến trình master đã chạy sau thời điểm nâng cấp:

```bash
ps -o pid,lstart,cmd -C php-fpm8.3 | head -3
```

Chỉ coi máy đã được vá khi phiên bản gói không thấp hơn phiên bản mà php.net công bố đã sửa, hoặc changelog của gói nhắc đến CVE cần vá:

```bash
apt changelog php8.3-fpm | head -40
```

## Bước 5: kiểm tra sau khi vá

- `curl -I` từng site trả `200` (hoặc đúng mã chuyển hướng), thời gian phản hồi bình thường.
- Không có lỗi mới trong `logs/php-error.log` và `logs/error.log` của từng site.
- Chạy một đơn hàng thử hoặc form liên hệ trên site có thương mại điện tử.
- Theo dõi tiến trình con của PHP-FPM trong 24 giờ đầu: tiến trình PHP sinh shell, `fsockopen`, `mkfifo`, `/dev/tcp/` là dấu hiệu đáng ngờ.

```bash
ps -eo pid,ppid,user,args --forest | grep -E 'php-fpm' -A3 | head -80
```

## Phiên bản hết hỗ trợ (7.4, 8.1 và cũ hơn)

Không có bản vá chính thức. Nâng nhánh (bước 2 và 3). Nếu chưa kịp: giới hạn truy cập qua CloudFlare WAF hoặc security group, hoặc tạm ngừng site.
Không chặn cổng 80 của cả máy; mọi site đều qua ALB nên làm vậy sẽ làm sập hết.

## Đợt vá khẩn cấp (CVE-2026-109618 và CVE-2026-109619)

PHP lên lịch phát hành 14:00 UTC ngày 21/10/2026 (21:00 giờ Việt Nam). Lúc viết tài liệu này, phiên bản bị ảnh hưởng và phiên bản đã vá chưa được công bố.

1. Trước giờ phát hành: bước 1 (kiểm kê), snapshot máy, bản sao lưu mới nhất ([runbook-backup-restore.md](runbook-backup-restore.md)).
2. Đọc [thông báo của php.net](https://www.php.net/archive/2026.php#2026-10-20-1) để lấy phiên bản bị ảnh hưởng và phiên bản vá.
3. Chờ gói của Ubuntu hoặc Ondrej PPA: `apt update && apt list --upgradable | grep php`. Gói chưa về thì chưa vá được.
4. Vá theo bước 4, kiểm tra theo bước 5.
5. Nhánh hết hỗ trợ: xử lý theo phần trên.

## Việc có thể tự động hoá sau này

`install-php-version.sh` (cài song song kèm đủ extension), một script kiểm kê phiên bản theo site, và một script vá có `--dry-run`. Chưa viết.
