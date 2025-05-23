---
id: adr-006-auth-strategy
title: ADR-006: Chiến lược xác thực người dùng (Authentication) cho hệ thống dx_vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [auth, security, dx_vas, oauth2, otp]
---

## 📌 Bối cảnh

Hệ thống dx_vas phục vụ nhiều nhóm người dùng:
- Giáo viên, nhân viên trường học (nội bộ VAS)
- Học sinh (sử dụng dịch vụ học tập)
- Phụ huynh (theo dõi kết quả học tập, nhận thông báo)

Google Workspace for Education hiện được cấp cho học sinh, giáo viên và nhân viên, nhưng **không cấp cho phụ huynh**. Do đó, cần có cơ chế xác thực phù hợp cho từng nhóm người dùng.

---

## 🧠 Quyết định

**Hệ thống dx_vas áp dụng cơ chế xác thực phân biệt theo nhóm người dùng:**
- OAuth2 (Google) cho học sinh, giáo viên, nhân viên nội bộ
- Email hoặc SĐT + OTP (Firebase hoặc custom OTP backend) cho phụ huynh

> Quyết định không embed `permissions` vào JWT access token để tránh tăng kích thước và rủi ro stale data. Các quyền được tra cứu từ Redis hoặc DB trong mỗi request nếu cần, và được forward dưới dạng `X-Permissions` từ API Gateway.

---

## 🔐 Chi tiết chiến lược xác thực

### 1. OAuth2 (Google Workspace Education) làm Identity Provider
- Áp dụng cho: **học sinh, giáo viên, nhân viên**
- Sử dụng OAuth2 standard flow
- Mỗi lần đăng nhập tạo JWT access token + refresh token
- Refresh token TTL: **30–90 ngày tùy loại user**, lưu Redis hoặc DB (theo [adr-024](./adr-024-data-anonymization-retention.md))

### 2. Xác thực phụ huynh (Không dùng Google)
- Áp dụng cho: **phụ huynh**
- Do không có Google Workspace Account → sử dụng **Email hoặc Phone + OTP**
- Tùy chọn provider:
  - Firebase Auth (email OTP, phone OTP)
  - Custom OTP API tích hợp Zalo OA hoặc SMS Gateway
- Sau xác thực OTP thành công → tạo JWT giống như OAuth2 (gắn claim `auth_method=otp`, `role=parent`)
- Refresh token TTL: **30–90 ngày tùy loại user** (quy định cụ thể cho `parent` sẽ được config theo chính sách nội bộ)

### 3. Common JWT Structure
```json
{
  "sub": "user_id",
  "email": "parent@example.com",
  "role": "parent",
  "auth_method": "otp",
  "iat": 1710000000,
  "exp": 1710003600
}
```
- `role`: chuỗi đơn (not array), phản ánh vai trò chính của user: `student`, `teacher`, `parent`, `admin`...
- `auth_method`: chỉ rõ phương thức xác thực, ví dụ: `oauth2`, `otp`, `internal`

---

## 📤 Frontend nhận token
- Frontend nhận access token từ backend sau khi xác thực thành công
- **Không nên lưu token trong localStorage**. Thay vào đó:
  - SPA: lưu trong `sessionStorage`
  - SSR hoặc frontend truyền thống: lưu trong `HttpOnly Cookie`
- **Frontend KHÔNG tự thêm các header `X-Role`, `X-User-ID`, `X-Permissions`**. Những header này sẽ được sinh ra bởi API Gateway sau khi JWT được xác thực.

---

## 🧩 Tích hợp RBAC & Service header forwarding
- API Gateway xác thực JWT, sau đó:
  - Tra `role` và `permissions` (từ DB hoặc Redis)
  - Forward các header sau:
```
X-User-ID: user_id
X-Role: parent
X-Auth-Method: otp
X-Permissions: read:score, receive:notification
```
- Các service backend có thể tin cậy vào các header này và phân quyền theo RBAC (ADR-007), không cần decode lại JWT.

---

## 🔐 Anonymous & Internal Service Auth
- Một số API public cho phép truy cập anonymous (có quota/rate-limit riêng)
- Các internal service (e.g. Notification Service, Cron) sử dụng service account JWT với `auth_method=internal`, `role=system` hoặc `cron` để phân biệt RBAC

---

## ✅ Lợi ích
- Phân tách rõ loại người dùng và hình thức xác thực
- Cho phép mở rộng dễ dàng thêm các provider khác
- Phụ huynh có thể sử dụng hệ thống mà không cần tạo tài khoản Google riêng

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| OTP spam hoặc brute force | Limit rate + CAPTCHA |
| Google OAuth2 token bị lộ | Chỉ chấp nhận domain `@truongvietanh.edu.vn` + refresh token có TTL rõ ràng |
| JWT reuse từ phụ huynh sang học sinh | Check `role`, `auth_method` strict ở API Gateway + audit |

---

## 📎 Tài liệu liên quan

- RBAC Strategy: [ADR-007](./adr-007-rbac.md)
- Security Hardening: [ADR-004](./adr-004-security.md)
- Secrets Management: [ADR-003](./adr-003-secrets.md)
- Audit Logging: [ADR-008](./adr-008-audit-logging.md)
- Token TTL & Data policy: [ADR-024](./adr-024-data-anonymization-retention.md)

---
> “Xác thực không chỉ là cánh cửa — đó là cách bạn kiểm soát ai bước vào hệ thống.”
