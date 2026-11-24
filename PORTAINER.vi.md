# PhoneInfoga — Cài qua Portainer Git (mạng LAN)

Dùng fork [hoangthang0106/phoneinfoga](https://github.com/hoangthang0106/phoneinfoga), nhánh **`portainer-lan`**; không thay đổi Compose gốc.
**Cổng 29004/TCP là đề xuất chưa kiểm tra trên VM/LXC thật.**

**Quy hoạch port chung:** [IT-MANAGER / sổ port](https://github.com/hoangthang0106/thang/blob/main/ports/PORT_REGISTRY.vi.md). Dải **29001–29099** dành riêng cho OSINT, số tiếp theo dự kiến cấp mới là **29006** (vẫn cần kiểm tra trực tiếp). Các port ứng dụng (5000/5001/8080/9005) chỉ tồn tại bên trong network Docker từng Stack, **không publish ra host và không xung đột** với container khác.

 Backend vẫn sử dụng cổng dịch vụ riêng 5000/TCP trong mạng Docker (không publish). Không đổi tùy tiện cổng lắng nghe của phần mềm.


## GHI CHÚ CÔNG DỤNG — ĐỌC TRÊN ĐIỆN THOẠI

- **Ứng dụng:** PhoneInfoga
- **Dùng để:** Tra cứu cấu trúc, quốc gia, nhà mạng và dấu vết số điện thoại từ nguồn công khai; không xác minh việc đăng ký Zalo/WhatsApp.
- **Port truy cập sau cài:** `http://LAN_IP:29004` (chỉ khi cổng trống và Stack chạy thành công).
- **Kiểu cài:** Image dựng sẵn: sundowndev/phoneinfoga:latest.
- **Nguyên tắc:** Kết quả OSINT chỉ là dấu hiệu; chỉ kiểm tra dữ liệu được phép. Không lưu email/số điện thoại, khóa API hoặc mật khẩu trong fork công khai.

### Thao tác Portainer bằng điện thoại

1. Chrome → Portainer → **Environments → Containers/Stacks**, chọn đúng Docker endpoint. Kiểm tra chưa có Stack này.
2. Vào **Stacks → Add stack → Repository**, nhập các giá trị Git bên dưới.
3. Kéo xuống **Environment variables** → **Add an environment variable** từng dòng. Nhập `LAN_IP`, `OSINT_AUTH_USER`, `OSINT_AUTH_PASSWORD` (mật khẩu tự đặt, không phải hash).
4. Kiểm tra host port 29004 không bị chiếm (cả container dừng). Sau đó mới **Deploy the stack**.
5. Mở **Containers → Logs** của gateway và ứng dụng. Nếu cả hai chạy bình thường, mở địa chỉ LAN trên điện thoại rồi đăng nhập bằng **mật khẩu gốc**.
6. Nếu build lỗi, xem phương án `portainer-image-only.yml` ở bên dưới; **không đổi cổng ngẫu nhiên để chữa lỗi build**.

## 1. Kiểm tra trước bằng lệnh chỉ đọc trên đúng VM/LXC Docker

```bash
hostname -I
docker ps -a --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
docker compose ls -a
sudo ss -lntup
sudo ss -ltnp '( sport = :29004 )'
docker ps -aq | xargs -r docker inspect --format '{{.Name}} {{json .HostConfig.PortBindings}}'
```

Không coi port trống nếu chưa xem listener và port được đặt cho container đang dừng. Nếu 29004 trùng, đặt `PHONEINFOGA_HOST_PORT` thành cổng TCP khác đã xác minh trống. Không dừng/xóa container cũ.

## 2. Tự đặt mật khẩu trong Portainer — không cần tạo hash

Trong Portainer → **Environment variables**, đặt `OSINT_AUTH_PASSWORD` là mật khẩu mạnh do bạn tự chọn. Container gateway **tự tạo mật khẩu tự nhập khi khởi động**; bạn không phải chạy `caddy hash-password` hay dán `OSINT_AUTH_PASSWORD`. Không nhập mật khẩu vào file GitHub hoặc nhắn qua chat. Người có quyền quản trị Portainer/Docker vẫn có thể xem biến môi trường. Dùng HTTPS nội bộ/VPN khi xử lý dữ liệu nhạy cảm, vì HTTP Basic Auth không mã hóa.

## 3. Portainer → Stacks → Add stack → Repository

- **Stack name:** `osint-phoneinfoga`
- **Repository URL:** `https://github.com/hoangthang0106/phoneinfoga`
- **Reference:** `refs/heads/portainer-lan`
- **Compose path:** `portainer-compose.yml`
- **Environment variables** (nhập tay trước Deploy):
  - `LAN_IP`: IP thật trên interface của VM/LXC đang chạy Docker, không mặc định dùng IP Proxmox host.
  - `OSINT_AUTH_USER`: username dùng để mở web UI (ví dụ `itadmin`).
  - `OSINT_AUTH_PASSWORD`: mật khẩu mạnh bạn tự đặt.
  - `PHONEINFOGA_HOST_PORT`: **tùy chọn**, chỉ nhập nếu cần đổi host port 29004.

**Sử dụng image dựng sẵn** `sundowndev/phoneinfoga:latest` thay vì build. Compose vẫn lưu trong fork của bạn; image không được build từ fork.

## 4. Kiểm tra sau khi Deploy (chỉ khi đã chủ động xác nhận)

- Đợi cả container ứng dụng và **gateway** Running, xem **Logs** nếu có lỗi.
- Mở **http://LAN_IP:29004** (thay port nếu đã đổi), nhập username và **mật khẩu gốc** bạn dùng tạo hash.
- Kiểm tra khả năng truy vấn dữ liệu bạn sở hữu hoặc được ủy quyền. Không có công cụ nào biết hết nền tảng mà một email/số điện thoại đã đăng ký.

## 5. Bảo mật & giới hạn

Gateway dùng Caddy `basic_auth`, backend không publish `ports:` ra host. Tuy nhiên **HTTP Basic Auth trên mạng LAN không mã hóa**; không mở cổng ra Internet, cần firewall giới hạn LAN; khi sử dụng dữ liệu nhạy cảm hãy chuyển sang HTTPS nội bộ. Người có quyền quản trị Portainer/Docker có thể xem biến môi trường. Không lưu khóa API, bí mật, dữ liệu nhạy cảm trong fork công khai.
PhoneInfoga chỉ tìm thông tin công khai/nhà mạng/khu vực từ số điện thoại, không chứng minh được số đó có đăng ký Zalo, WhatsApp hay mọi tài khoản.
Dữ liệu ở Docker named volume (nếu có) không tự được backup. Không xóa volume hoặc bật tự cập nhật mà chưa kiểm tra. **Đây chỉ là cấu hình trên GitHub, không phải xác nhận máy chủ đã triển khai.**
