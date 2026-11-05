# Bài 6: Quy trình bảo mật mã nguồn trước khi đẩy lên GitHub Public

## 1. Bối cảnh & Yêu cầu chức năng
* **Bối cảnh**: Team của bạn chuẩn bị đẩy mã nguồn dự án thương mại điện tử (E-commerce) lên **GitHub Public** để đối tác kiểm tra.
* **Yêu cầu chức năng**:
  1. Xây dựng **Pre-push Checklist** để quét sạch các API Keys bị hardcode.
  2. Thiết lập cấu trúc tệp **`.env.example`** chuyên nghiệp cho các môi trường Dev, Test, và Production.
  3. Hướng dẫn team kích hoạt **2FA bắt buộc** cho mọi tài khoản tham gia repository.

## 2. Pre-push Checklist: Quét sạch API Keys và dữ liệu nhạy cảm

| Bước kiểm tra | Hành động cụ thể cần thực hiện trước khi `git push` | Lý do & Lưu ý quan trọng |
| :--- | :--- | :--- |
| **1. Cấu hình `.gitignore` chuẩn chỉnh** | • Đưa toàn bộ tệp chứa biến môi trường và chứng chỉ bảo mật vào `.gitignore`:<br>  - `.env`, `.env.local`, `.env.development`, `.env.test`, `.env.production`<br>  - `*.pem`, `*.key`, `*.p12`, `serviceAccountKey.json`<br>• Giữ lại tệp mẫu bằng quy tắc ngoại lệ: `!.env.example` | Ngăn Git theo dõi (track) và đẩy nhầm tệp cấu hình thật từ máy cá nhân lên kho mã nguồn công khai. |
| **2. Rà soát và xóa Hardcode trong mã nguồn** | • Dùng công cụ tìm kiếm (Regex Search) quét toàn bộ mã nguồn các từ khóa: `API_KEY`, `SECRET`, `PASSWORD`, `PRIVATE_KEY`, `Bearer `, `sk_live_`, `mongodb+srv://`, `postgres://`.<br>• Thay thế toàn bộ chuỗi khóa cố định bằng lời gọi biến môi trường (ví dụ: `process.env.STRIPE_SECRET_KEY`). | Đảm bảo không còn bất kỳ chuỗi khóa thật nào nằm trực tiếp trong các tệp `.js`, `.ts`, `.py`, `.java` hoặc tệp cấu hình Docker/CI. |
| **3. Quét tự động bằng công cụ Secret Scanner** | • Chạy công cụ quét chuyên dụng như **Gitleaks**, **TruffleHog** hoặc **git-secrets** trên toàn bộ kho mã nguồn:<br>  `gitleaks detect --source . --verbose`<br>• Cài đặt **Pre-commit / Pre-push Hook** để tự động chặn lệnh commit/push nếu phát hiện chuỗi có độ ngẫu nhiên cao giống API Key. | Phát hiện chính xác các khóa bị bỏ sót bằng cả mẫu nhận diện (Regex) lẫn đo độ hỗn loạn chuỗi (Entropy Analysis). |
| **4. Kiểm tra và làm sạch toàn bộ lịch sử Git (`git log`)** | • Kiểm tra xem trong các commit quá khứ có từng lỡ commit tệp `.env` hay API Key nào không.<br>• **Nếu từng lỡ commit**: Không được chỉ xóa tệp rồi tạo commit mới (vì khóa vẫn nằm nguyên trong lịch sử `git log`). Phải dùng **`git-filter-repo`** hoặc **BFG Repo-Cleaner** để xóa sạch dấu vết trong lịch sử Git, đồng thời **hủy (Revoke) và tạo lại (Rotate) toàn bộ API Key đó ngay lập tức**. | Bot tự động của tin tặc quét toàn bộ lịch sử commit của mọi kho GitHub Public chỉ trong vài giây sau khi vừa mở công khai. |

## 3. Thiết lập cấu trúc tệp `.env.example` chuyên nghiệp (Dev, Test, Production)

Tệp `.env.example` được đẩy lên GitHub để hướng dẫn lập trình viên và đối tác biết dự án cần khai báo những biến nào. **Nguyên tắc vàng**: Chỉ ghi tên biến và giá trị mẫu minh họa, tuyệt đối không điền mật khẩu hay khóa thật.

```ini
# ==============================================================================
# CẤU HÌNH MÔI TRƯỜNG MẪU (.env.example) - DỰ ÁN THƯƠNG MẠI ĐIỆN TỬ
# Hướng dẫn: Sao chép tệp này thành .env.development, .env.test hoặc .env.production
# và điền giá trị thực tế tương ứng với từng môi trường. KHÔNG commit tệp .env thật!
# ==============================================================================

# ------------------------------------------------------------------------------
# 1. ĐỊNH DANH MÔI TRƯỜNG & MÁY CHỦ ỨNG DỤNG
# - Dev : NODE_ENV=development | APP_URL=http://localhost:3000
# - Test: NODE_ENV=test        | APP_URL=https://staging.example-shop.com
# - Prod: NODE_ENV=production  | APP_URL=https://example-shop.com
# ------------------------------------------------------------------------------
NODE_ENV=development
PORT=3000
APP_URL=http://localhost:3000
LOG_LEVEL=debug

# ------------------------------------------------------------------------------
# 2. CƠ SỞ DỮ LIỆU & BỘ NHỚ ĐỆM (DATABASE & REDIS)
# - Dev : Dùng DB cài đặt cục bộ (Docker/Localhost)
# - Test: Dùng DB riêng cho kiểm thử tự động (Tự động xóa dữ liệu sau khi chạy test)
# - Prod: Dùng Managed Database có bật SSL và giới hạn IP truy cập
# ------------------------------------------------------------------------------
DB_HOST=localhost
DB_PORT=5432
DB_NAME=ecommerce_dev_db
DB_USER=your_db_username_here
DB_PASSWORD=your_strong_db_password_here
DB_SSL_ENABLED=false

REDIS_URL=redis://localhost:6379

# ------------------------------------------------------------------------------
# 3. XÁC THỰC & BẢO MẬT PHIÊN (AUTHENTICATION & JWT)
# Lưu ý: Sinh chuỗi ngẫu nhiên tối thiểu 64 ký tự bằng lệnh: openssl rand -hex 32
# Mỗi môi trường (Dev / Test / Prod) BẮT BUỘC dùng khóa bí mật hoàn toàn khác nhau!
# ------------------------------------------------------------------------------
JWT_ACCESS_SECRET=replace_with_random_64_character_hex_string
JWT_REFRESH_SECRET=replace_with_another_random_64_character_hex_string
JWT_EXPIRES_IN=15m

# ------------------------------------------------------------------------------
# 4. CỔNG THANH TOÁN & DỊCH VỤ BÊN THỨ BA (THIRD-PARTY INTEGRATIONS)
# - Dev & Test: Bắt buộc dùng khóa Sandbox / Test (ví dụ: sk_test_..., pk_test_...)
# - Production: Chỉ nạp khóa Live (sk_live_...) trực tiếp trên trình quản lý Secret của Server
# ------------------------------------------------------------------------------
STRIPE_PUBLIC_KEY=pk_test_your_stripe_public_key_here
STRIPE_SECRET_KEY=sk_test_your_stripe_secret_key_here
STRIPE_WEBHOOK_SECRET=whsec_your_webhook_signing_secret_here

AWS_REGION=ap-southeast-1
AWS_S3_BUCKET_NAME=ecommerce-assets-dev
AWS_ACCESS_KEY_ID=your_aws_access_key_id_here
AWS_SECRET_ACCESS_KEY=your_aws_secret_access_key_here

SMTP_HOST=smtp.mailtrap.io
SMTP_PORT=2525
SMTP_USER=your_smtp_username_here
SMTP_PASS=your_smtp_password_here
```

## 4. Hướng dẫn kích hoạt 2FA bắt buộc cho mọi tài khoản tham gia Repository

| Đối tượng thực hiện | Các bước thao tác chi tiết trên GitHub |
| :--- | :--- |
| **Phần A: Dành cho từng thành viên trong Team (Bật 2FA cá nhân trước)** | 1. Đăng nhập vào GitHub $\rightarrow$ Nhấn vào ảnh đại diện ở góc trên bên phải $\rightarrow$ Chọn **Settings**.<br>2. Ở thanh menu bên trái, mục *Access*, chọn **Password and authentication**.<br>3. Tại phần **Two-factor authentication**, nhấn nút **Enable two-factor authentication**.<br>4. **Thiết lập phương thức xác thực (Ưu tiên tuyệt đối)**:<br>   • **Cách 1 (Ứng dụng TOTP)**: Dùng *Bitwarden Authenticator*, *Google Authenticator* hoặc *1Password* quét mã QR hiển thị trên màn hình rồi nhập mã 6 chữ số để xác nhận.<br>   • **Cách 2 (Passkeys / Khóa phần cứng)**: Đăng ký khóa bảo mật chuẩn FIDO2/WebAuthn (như YubiKey hoặc cảm biến vân tay thiết bị).<br>5. **Lưu mã khôi phục (Recovery Codes)**: Tải tệp Recovery Codes về và lưu ngay vào kho mã hóa của Password Manager (tuyệt đối không bỏ qua bước này). |
| **Phần B: Dành cho Quản trị viên (Bắt buộc 2FA toàn Organization / Repository)** | 1. Đảm bảo tất cả thành viên trong nhóm đã hoàn thành **Phần A** (vì bất kỳ ai chưa bật 2FA sẽ bị GitHub tự động trục xuất khỏi Organization ngay khi bật quy tắc bắt buộc).<br>2. Truy cập vào trang **GitHub Organization** quản lý dự án $\rightarrow$ Chọn tab **Settings**.<br>3. Ở menu bên trái, mục *Security*, nhấn vào **Authentication security**.<br>4. Tại mục **Two-factor authentication**, tích chọn ô:<br>   ☑ **"Require two-factor authentication for everyone in the organization"**.<br>5. Nhấn **Save** và nhập lại mật khẩu/mã 2FA của Quản trị viên để xác nhận áp dụng chính sách. |
