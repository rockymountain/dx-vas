# 📘 Interface Contract – Auth Service (dx\_vas)

## 🧭 Mục tiêu

Tài liệu này định nghĩa Interface Contract cho **Auth Service**, chịu trách nhiệm:

* Xác thực người dùng (OTP login, Google OAuth)
* Phát hành và làm mới JWT
* Hỗ trợ đăng xuất, kiểm tra trạng thái token
* Phục vụ đa kênh: Web, Mobile App, API Public (nếu cần)

---

## 🧩 Endpoint Auth Service cung cấp

| Method | Endpoint                | Mô tả                                      | Input Schema            | Output Schema      |
| ------ | ----------------------- | ------------------------------------------ | ----------------------- | ------------------ |
| POST   | `/auth/request-otp`     | Gửi OTP đến email/số điện thoại            | `OtpRequest`            | `OtpRequestResult` |
| POST   | `/auth/verify-otp`      | Xác thực mã OTP và phát hành JWT           | `OtpVerify`             | `TokenPair`        |
| GET    | `/auth/google-login`    | Bắt đầu Google OAuth2 (redirect)           | `redirect_uri: str`     | — (HTTP Redirect)  |
| POST   | `/auth/google-callback` | Xử lý mã trả về từ Google và phát hành JWT | `GoogleCallbackPayload` | `TokenPair`        |
| POST   | `/auth/refresh`         | Làm mới access token bằng refresh token    | `RefreshRequest`        | `TokenPair`        |
| POST   | `/auth/logout`          | Thu hồi refresh token (đăng xuất)          | `LogoutRequest`         | `LogoutResult`     |

---

## 🔐 Xác thực & Phân quyền

* Các endpoint `/auth/*` không yêu cầu access token
* Sau khi xác thực thành công, Auth Service trả về:

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "expires_in": 3600
}
```

* `access_token` là JWT dùng gọi API Gateway
* `refresh_token` được quản lý nội bộ hoặc lưu trong cookie (mobile, frontend tuỳ luồng)

---

## 📥 Input Schema tiêu biểu

### OtpRequest

```json
{
  "identity": "parent@example.com",
  "channel": "email" // hoặc "sms"
}
```

### OtpVerify

```json
{
  "identity": "parent@example.com",
  "otp": "123456"
}
```

### GoogleCallbackPayload

```json
{
  "code": "auth_code_abc",
  "redirect_uri": "https://client.dx.edu.vn/oauth2/callback"
}
```

> `redirect_uri` phải khớp chính xác với cấu hình của hệ thống để ngăn giả mạo (ADR-004).

### RefreshRequest

```json
{
  "refresh_token": "..."
}
```

### LogoutRequest

```json
{
  "refresh_token": "..."
}
```

---

## 📤 Output Schema tiêu biểu

### TokenPair

```json
{
  "access_token": "...",
  "refresh_token": "...",
  "expires_in": 3600
}
```

### OtpRequestResult

```json
{
  "channel": "email",
  "message_id": "otp-xyz-123"
}
```

### LogoutResult

```json
{
  "success": true
}
```

> Một số hệ thống có thể trả HTTP 204 No Content cho logout. Tại dx\_vas, dùng JSON rõ ràng để tiện logging và frontend xử lý.

---

## ⚠️ Lỗi đặc thù

| Code                      | Mô tả                                                |
| ------------------------- | ---------------------------------------------------- |
| `OTP_INVALID`             | Mã OTP không hợp lệ *(đã bao gồm hết hạn)*           |
| `IDENTITY_NOT_FOUND`      | Email/số điện thoại chưa đăng ký trong hệ thống      |
| `REFRESH_TOKEN_EXPIRED`   | Refresh token không còn hợp lệ                       |
| `GOOGLE_CALLBACK_INVALID` | Google callback thất bại hoặc bị giả mạo             |
| `USER_ACCOUNT_LOCKED`     | Tài khoản bị khoá, không cho phép login              |
| `USER_ACCOUNT_INACTIVE`   | Tài khoản chưa được kích hoạt hoặc đã bị vô hiệu hoá |

> Tất cả lỗi đều tuân theo [ADR-011](../ADR/adr-011-api-error-format.md). Các mã lỗi nên được chuẩn hoá và không lặp lại nếu đã tồn tại trong ADR-011.

---

## ✅ Ghi chú

* Auth Service có thể gắn thêm captcha nếu hệ thống phát hiện spam OTP (theo [ADR-004](../ADR/adr-004-security.md))
* Refresh token hỗ trợ rotation nếu triển khai nâng cao
* Google login không phát hành tài khoản mới – chỉ liên kết với người dùng đã có trong hệ thống
* Auth Service không cần permission RBAC – chỉ phát hành token, còn quyền sẽ được evaluate tại API Gateway
* Mọi response tuân thủ cấu trúc [ADR-012](../ADR/adr-012-response-structure.md):

```json
{
  "data": {...},
  "error": null,
  "meta": {
    "timestamp": "...",
    "trace_id": "...",
    "source": "auth_service"
  }
}
```

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-004: Security Strategy](../ADR/adr-004-security.md)
* [ADR-011: API Error Format](../ADR/adr-011-api-error-format.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)

---

> “Auth Service là cánh cửa đầu tiên – vừa an toàn, vừa đơn giản, vừa mở rộng được cho mọi kênh.”
