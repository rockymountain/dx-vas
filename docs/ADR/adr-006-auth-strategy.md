---
id: adr-006-auth-strategy
title: ADR-006: Chiến lược Xác thực người dùng (Authentication Strategy) cho hệ thống dx_vas
status: accepted
author: DX VAS Security Team
date: 2025-06-22
tags: [auth, oauth2, jwt, identity, dx_vas, gateway]
---

## 📌 Bối cảnh

Hệ thống **dx_vas** phục vụ nhiều nhóm người dùng:
- Học sinh, phụ huynh, giáo viên (qua Portal/LMS)
- Nhân viên nhà trường (CRM, SIS, quản trị)
- Dịch vụ nội bộ gọi API qua Gateway hoặc service-to-service

Tất cả request đều đi qua API Gateway và các dịch vụ phía sau (CRM Adapter, LMS Proxy...). Việc xác thực người dùng và dịch vụ phải:
- Đảm bảo an toàn, đơn giản, có thể scale
- Dễ tích hợp với frontend, bên thứ ba (SSO, OAuth2)
- Cho phép phát hành và xác minh token tiêu chuẩn (JWT)

## 🧠 Quyết định

**Áp dụng chiến lược xác thực tập trung qua API Gateway sử dụng Google OAuth2, phát hành JWT access token ngắn hạn và refresh token an toàn, hỗ trợ lưu session frontend bằng cookie HttpOnly hoặc sessionStorage.**

## 🔐 Kiến trúc xác thực

### 1. OAuth2 (Google) làm Identity Provider
- Frontend (Portal, Admin Webapp) sẽ redirect tới Google để xác thực
- Sau khi đăng nhập thành công, API Gateway sẽ:
  - Xác minh `id_token`
  - Tạo JWT access token (`15 phút`) chứa `sub`, `email`, `role`, `permissions`
  - Tạo refresh token lưu trong DB hoặc Redis (với TTL `30–90 ngày` tùy loại user)

### 2. Frontend nhận token
- Có 2 cơ chế:
  - **SPA (JS):** lưu access token trong `sessionStorage`, refresh token không lưu ở client → call `/auth/refresh` định kỳ
  - **SSR (Next.js):** lưu token trong `HttpOnly cookie`, auto send qua header `Cookie`

### 3. Token format (JWT)
```json
{
  "sub": "user_123",
  "email": "abc@gmail.com",
  "role": "student",
  "permissions": ["student.read", "grades.view"],
  "exp": 1710000000
}
```
- Ký bằng `HS256` hoặc `ES256`
- Secret/key được rotate định kỳ (xem [ADR-003](./adr-003-secrets.md))

### 4. RBAC và Header forwarding
- API Gateway phân quyền theo `role` và `permissions` đã embed trong JWT
- Forward các header chuẩn:
  - `X-User-Id`
  - `X-Role`
  - `X-Permissions`
  - `X-Token-Exp`
- Các backend service chỉ cần verify lại chữ ký JWT (nếu không tin gateway), hoặc tin `header` nếu trust gateway

### 5. Anonymous & internal service
- Một số endpoint public không cần xác thực (VD: `/public/school-info`)
- Service-to-service dùng JWT ký bởi secret riêng, hoặc xác thực bằng service account với mTLS hoặc WIF

## 🛡️ Bảo mật
- Access token hết hạn sau 15 phút → cần refresh
- Refresh token revoke bất cứ lúc nào (offboarding, logout)
- IP lạ / thiết bị mới → yêu cầu re-auth
- Middleware kiểm tra token ở tất cả API sensitive

## ✅ Lợi ích

- Dễ dùng với Google Workspace sẵn có của nhà trường
- Không cần quản lý mật khẩu nội bộ → giảm rủi ro
- Token-based → dễ tích hợp, mở rộng, stateless
- Cho phép frontend chủ động kiểm soát phiên
- Phân quyền linh hoạt qua JWT hoặc Redis cache

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Refresh token bị đánh cắp | Lưu server-side (DB/Redis), invalidate theo IP/User Agent |
| JWT bị dùng quá hạn | TTL ngắn, có check `exp`, middleware reject |
| Trộn giữa session-based và token-based | Tách rõ Auth flow cho SPA vs SSR |

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Tự quản lý user/password | Tốn công bảo trì, không bảo mật bằng Google SSO |
| OAuth toàn bộ qua frontend | Khó kiểm soát token issuance, không thống nhất flow |
| Dùng API Key truyền tay | Không kiểm soát được mức độ truy cập, không rotate được |

## 📎 Tài liệu liên quan

- RBAC Strategy: [ADR-007](./adr-007-rbac.md)
- Security Hardening: [ADR-004](./adr-004-security.md)
- Secrets: [ADR-003](./adr-003-secrets.md)

---
> “Xác thực tốt không chỉ bảo vệ hệ thống – mà còn tạo trải nghiệm đăng nhập liền mạch cho người dùng.”
