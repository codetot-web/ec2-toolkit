# Runbook: sao lưu và khôi phục

Chi tiết từng lệnh nằm trong [../bash-scripts/BACKUP-SETUP.md](../bash-scripts/BACKUP-SETUP.md). Trang này là bản tóm tắt cho người vận hành và danh sách kiểm tra.

## Chiến lược

| Thành phần      | Cách làm                                                                                                       |
| --------------- | -------------------------------------------------------------------------------------------------------------- |
| Database        | `wp db export` nén gzip, tải lên S3 với tên có mốc thời gian. Vòng đời S3 lo việc lưu trữ và hết hạn           |
| Uploads         | `aws s3 sync` tăng dần. `--delete` bị tắt có chủ đích: file xoá cục bộ vẫn còn trên S3, versioning giữ lịch sử |
| Bản dump cục bộ | Giữ 7 ngày trong `backups/` của site (`--retention-days`)                                                      |
| Xác thực        | IAM role của EC2, không có khoá AWS trong script hay môi trường                                                |
| Chạy bởi        | user `ubuntu` qua cron, không cần `sudo`                                                                       |

Script này không thay thế sao lưu tự động của RDS. Muốn "khôi phục về 14:23 hôm qua" thì dùng point-in-time recovery của RDS.

## Cài đặt (một lần)

| Pha | Việc                                                                           | Chi tiết                 |
| --- | ------------------------------------------------------------------------------ | ------------------------ |
| A   | Tạo bucket, bật versioning, mã hoá, chặn truy cập công khai, đặt lifecycle     | BACKUP-SETUP.md, Phase A |
| B   | Tạo IAM policy và gắn vào role của EC2, kiểm tra `aws sts get-caller-identity` | Phase B                  |
| C   | Cài AWS CLI v2 (x86_64 hoặc ARM)                                               | Phase C                  |
| D   | Cài `backup-site.sh` và chạy thử bằng tay                                      | Phase D                  |
| E   | Lên lịch cron, logrotate, tuỳ chọn DB hằng ngày                                | Phase E                  |
| F   | Kiểm tra sau một tuần, báo động CloudWatch khi thiếu bản sao                   | Phase F                  |

Không bắt đầu pha D khi pha B chưa qua.

## Chạy tay

```bash
backup-site --site=acmeshop --bucket=your-backups-prod
backup-site --site=acmeshop --bucket=your-backups-prod --skip-uploads     # chỉ DB
backup-site --site=acmeshop --bucket=your-backups-prod --skip-db          # chỉ uploads
```

Cờ khác: `--prefix` (mặc định `backups/<site>`), `--retention-days`, `--storage-class` (mặc định `STANDARD_IA`), `--region`.

## Lịch cron mẫu (crontab của ubuntu)

```cron
# Chủ nhật 02:00, đầy đủ
0 2 * * 0 /usr/local/bin/backup-site --site=acmeshop --bucket=your-backups-prod >> /home/ubuntu/webapps/acmeshop/logs/backup.log 2>&1
# Hằng ngày 03:00, chỉ DB
0 3 * * * /usr/local/bin/backup-site --site=acmeshop --bucket=your-backups-prod --skip-uploads >> /home/ubuntu/webapps/acmeshop/logs/backup.log 2>&1
```

Nhiều site: so le ít nhất 30 phút. Cron phải đặt trong crontab của `ubuntu`, không phải `root`.

## Danh sách kiểm tra hằng tuần

- [ ] Bản dump DB mới nhất có trên S3 và có dung lượng hợp lý
- [ ] Số đối tượng uploads trên S3 xấp xỉ số file cục bộ
- [ ] `backups/` cục bộ không phình ra
- [ ] Dòng cuối của `backup.log` không có lỗi

```bash
aws s3 ls s3://your-backups-prod/backups/acmeshop/db/ | sort | tail -3
tail -20 /home/ubuntu/webapps/acmeshop/logs/backup.log
```

## Khôi phục

Thử khôi phục ít nhất một lần lên EC2 staging, trước khi cần dùng thật.

### Database (thao tác phá huỷ: xoá bảng hiện có)

```bash
aws s3 ls s3://your-backups-prod/backups/acmeshop/db/ | sort
aws s3 cp s3://your-backups-prod/backups/acmeshop/db/db-2026-05-04T19-00-00Z.sql.gz /tmp/restore.sql.gz
cd /home/ubuntu/webapps/acmeshop/public
gunzip -c /tmp/restore.sql.gz | wp db import -
wp cache flush
wp rewrite flush
rm /tmp/restore.sql.gz
```

Trước khi nhập, chụp một bản dump của trạng thái hiện tại: `wp db export /tmp/before-restore.sql`.

### Uploads

```bash
aws s3 sync s3://your-backups-prod/backups/acmeshop/uploads/ /home/ubuntu/webapps/acmeshop/public/wp-content/uploads/
```

Áp lại quyền sau đó (xem [troubleshooting.md](troubleshooting.md), mục quyền file).

### Một file bị xoá

```bash
aws s3api list-object-versions --bucket your-backups-prod --prefix backups/acmeshop/uploads/2024/03/important-photo.jpg
aws s3api get-object --bucket your-backups-prod --key backups/acmeshop/uploads/2024/03/important-photo.jpg --version-id <VERSION_ID> /tmp/important-photo.jpg
```

### Thảm hoạ (mất RDS)

Dựng RDS mới, tạo lại database và user bằng `create-site.sh` (tự đồng bộ mật khẩu), rồi nhập bản dump mới nhất theo các bước trên.

## Sự cố

| Triệu chứng                     | Nguyên nhân thường gặp                               | Xử lý                                             |
| ------------------------------- | ---------------------------------------------------- | ------------------------------------------------- |
| Không có thông tin xác thực AWS | Chưa gắn IAM role, hoặc sai vùng                     | `aws sts get-caller-identity`                     |
| `Access Denied` khi `PutObject` | Policy thiếu hoặc sai tên bucket                     | Kiểm tra lại policy ở Phase B                     |
| `wp db export failed`           | Sai mật khẩu trong wp-config, hoặc RDS không với tới | `wp db check`; kiểm tra security group của RDS    |
| Cron không chạy                 | Sai crontab hoặc cron tắt                            | `systemctl status cron`; `crontab -l` bằng ubuntu |
| File khoá còn sót sau khi kill  | Lần chạy trước chết giữa chừng                       | `rm /tmp/wp-backup-<site>.lock`                   |
| S3 sync rất chậm                | Nhiều file nhỏ                                       | Bình thường, phần liệt kê chiếm nhiều thời gian   |
