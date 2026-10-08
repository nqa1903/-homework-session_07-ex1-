# Bài 1: Quản lý người dùng giới hạn và truyền tải dữ liệu qua SFTP trên Windows

## 1. Mục tiêu

- Tạo tài khoản người dùng giới hạn `sftp-user` (không có quyền sudo) chuyên dùng để tải file log.
- Cấp quyền đọc tệp `/var/log/app-backup/backup-check.log` cho `sftp-user`.
- Dùng SFTP Client trên Windows (WinSCP) để kết nối và tải tệp log về máy cá nhân.

## 2. Môi trường

| Thành phần | Chi tiết |
|---|---|
| Máy chủ | VPS Ubuntu 22.04 |
| Địa chỉ VPS | `<IP_VPS>` |
| Cổng SSH | `<22 hoặc cổng đã đổi, ví dụ 2222>` |
| SFTP Client | WinSCP trên Windows |
| Tài khoản kết nối | `sftp-user` (xác thực bằng mật khẩu) |

## 3. Các bước thực hiện trên VPS

### 3.1. Tạo người dùng `sftp-user` với mật khẩu mạnh

```bash
sudo adduser sftp-user
```

Mật khẩu được đặt dài trên 16 ký tự, gồm chữ hoa, chữ thường, số và ký tự đặc biệt. Người dùng **không** được thêm vào nhóm `sudo`.

### 3.2. Tạo thư mục và tệp log giả lập

```bash
sudo mkdir -p /var/log/app-backup/
sudo touch /var/log/app-backup/backup-check.log
sudo bash -c 'echo "Backup status: SUCCESS at $(date)" > /var/log/app-backup/backup-check.log'
```

### 3.3. Cấp quyền đọc cho `sftp-user`

```bash
sudo chown -R root:sftp-user /var/log/app-backup
sudo chmod 750 /var/log/app-backup
sudo chmod 640 /var/log/app-backup/backup-check.log
```

Giải thích phân quyền:

| Đối tượng | Thư mục `/var/log/app-backup` (750) | Tệp `backup-check.log` (640) |
|---|---|---|
| Chủ sở hữu `root` | đọc, ghi, truy cập | đọc, ghi |
| Nhóm `sftp-user` | đọc, truy cập | chỉ đọc |
| Người khác | không có quyền | không có quyền |

Nhờ vậy `sftp-user` chỉ đọc được file log, không sửa hay xóa được.

### 3.4. Cho phép đăng nhập bằng mật khẩu cho riêng `sftp-user` (nếu server đang tắt)

Kiểm tra cấu hình hiện tại:

```bash
sudo sshd -T | grep -Ei 'passwordauthentication|^port|allowusers'
```

Nếu `passwordauthentication no`, thêm khối sau vào **cuối** `/etc/ssh/sshd_config` để chỉ bật mật khẩu cho `sftp-user`, các user khác vẫn dùng SSH key:

```
Match User sftp-user
    PasswordAuthentication yes
```

Kiểm tra cú pháp và nạp lại dịch vụ:

```bash
sudo sshd -t && sudo systemctl reload ssh
```

## 4. Kiểm tra trên VPS

```bash
id sftp-user
ls -l /var/log/app-backup/backup-check.log
sudo -u sftp-user cat /var/log/app-backup/backup-check.log
sha256sum /var/log/app-backup/backup-check.log
```

Kết quả (đối chiếu với output thực tế của tôi trong ảnh bên dưới):

- `id sftp-user` chỉ có nhóm `sftp-user`, **không có nhóm `sudo`**.
- `ls -l` hiển thị `-rw-r----- 1 root sftp-user ... backup-check.log`.
- `sudo -u sftp-user cat ...` đọc được nội dung file log.
  
## 5. Kết nối SFTP từ Windows bằng WinSCP

1. Mở WinSCP, chọn **New Session**.
2. Điền thông tin:
   - **File protocol:** SFTP
   - **Host name:** `<IP_VPS>`
   - **Port number:** `<cổng SSH>`
   - **User name:** `sftp-user`
   - **Password:** mật khẩu của `sftp-user`
3. Nhấn **Login**, chọn **Yes** khi WinSCP hỏi xác nhận host key.

4. Sau khi kết nối thành công, ở khung bên phải (Remote) truy cập thư mục `/var/log/app-backup/`.


5. Kéo thả tệp `backup-check.log` sang khung bên trái (Local) để tải về máy.


## 6. Kiểm tra trên Windows

Mở tệp vừa tải về và so sánh nội dung với file gốc trên VPS.


Kiểm tra toàn vẹn bằng hash SHA-256 (PowerShell):

```powershell
Get-FileHash .\backup-check.log -Algorithm SHA256
```

Giá trị hash trên Windows trùng khớp với kết quả `sha256sum` trên VPS, chứng tỏ tệp được truyền nguyên vẹn.


## 7. Kết luận

- Đã tạo thành công người dùng `sftp-user` có mật khẩu mạnh và không thuộc nhóm `sudo`.
- Đã tạo `/var/log/app-backup/backup-check.log` và cấp quyền chỉ đọc cho `sftp-user` theo nguyên tắc đặc quyền tối thiểu.
- Đã kết nối SFTP từ Windows bằng WinSCP và tải thành công tệp log về máy cá nhân, nội dung khớp với file gốc trên VPS.
