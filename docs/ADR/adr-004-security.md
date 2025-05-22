---
id: adr-004-security
title: ADR-004: Chiến lược Bảo mật tổng thể cho hệ thống dx_vas
status: accepted
author: DX VAS Security Team
date: 2025-06-22
tags: [security, hardening, dx_vas, cloud-run, jwt, rate-limiting]
---

## 📌 Bối cảnh

Hệ thống **dx_vas** gồm nhiều thành phần xử lý dữ liệu nhạy cảm như:
- Học sinh, giáo viên, điểm số, học phí (qua SIS, CRM, LMS)
- Dữ liệu cá nhân và hành vi người dùng (qua frontend và API Gateway)
- Giao tiếp với bên thứ ba (Zalo, OAuth, đối tác CRM)

Mọi giao tiếp đều đi qua các tầng public như Cloud Run, Redis, frontend web, API Gateway… nên cần thiết kế **chiến lược bảo mật toàn diện, đa lớp**.

## 🧠 Quyết định

**Áp dụng chiến lược security hardening đa tầng (transport, application, token, headers, logging, dependency, CI/CD), theo chuẩn OWASP và best practice của Google Cloud.**

## 🔐 Các lớp bảo vệ

### 1. Transport Layer
- Bắt buộc HTTPS trên tất cả Cloud Run endpoint
- Dùng TLS 1.2+ với cert của Google Managed
- Chặn HTTP fallback nếu dùng Cloud Load Balancer

### 2. Application Layer
- Sanitize input toàn bộ qua Pydantic / frontend validator
- Giới hạn kích thước request (body, file upload)
- Từ chối content-type không hợp lệ
- Dùng CSRF token nếu có cookie-based auth (VD: frontend admin panel có form)

### 3. Token & Session
- Sử dụng JWT với expiry ngắn (`15 phút`)
- Refresh token lưu trong Secret Manager hoặc DB, mã hoá trước khi lưu
- Frontend không dùng `localStorage` để lưu token → ưu tiên `HttpOnly cookie` hoặc `sessionStorage`
- Ký JWT bằng `HS256` hoặc `ES256` với secret/key được rotate định kỳ (xem [adr-003-secrets.md](./adr-003-secrets.md))

### 4. Rate Limiting & DoS protection
- Áp dụng rate limit theo user_id/IP bằng Redis (`slowapi` middleware)
- Cloud Armor bảo vệ ở layer IP + Geo-based block
- Có logic phân biệt giữa role admin/internal và client
- Alert nếu rate limit trả 429 > 5% trong 5 phút

### 5. Security Headers (qua Gateway / frontend)
- `Strict-Transport-Security`
- `X-Frame-Options: DENY`
- `X-Content-Type-Options: nosniff`
- `Referrer-Policy: no-referrer`
- `Permissions-Policy: geolocation=()`
- (Optional nếu HTML render): `Content-Security-Policy`

### 6. Dependency Management
- Lock phiên bản deps (`requirements.txt`, `package-lock.json`, `poetry.lock`)
- Scan bảo mật CI bằng `safety`, `bandit`, `npm audit`, `trivy`
- Alert nếu có CVE nặng qua Dependabot hoặc GitHub CodeQL

### 7. Logging & Monitoring
- Tuyệt đối không log access token/refresh token
- Log structured có `request_id`, `user_id`, `path`, `latency`, `status`
- Dùng masking với regex `.*(token|secret|key).*` để tránh rò rỉ
- Alert khi có:
  - Đăng nhập thất bại nhiều lần
  - Truy cập trái phép RBAC
  - Tăng đột biến status 403/429/5xx

### 8. CI/CD Pipeline
- Tất cả secrets qua `GitHub Secrets` và `Secret Manager`
- Pre-commit kiểm tra `TODO`, `print()`, debug, file `.env`
- Mọi PR phải pass lint + test + scan
- Production deploy yêu cầu approval manual

## ✅ Lợi ích

- Bảo vệ toàn bộ dữ liệu học sinh, người dùng khỏi rò rỉ hoặc tấn công
- Đáp ứng yêu cầu bảo mật nội bộ, audit, và tiêu chuẩn ngành
- Phát hiện sớm và ngăn chặn hành vi đáng ngờ
- Đồng bộ hóa bảo mật giữa backend, frontend và hạ tầng

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Bỏ sót layer security trong service mới | Template `security.md` + checklist bắt buộc CI |
| Token bị leak qua log/debug | Middleware masking + test CI block log lỗi chứa sensitive data |
| Config bảo mật bị override bởi dev | Apply default via Terraform/Env, enforce trong review CI |

## 🔄 Các lựa chọn đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Không enforce rate limit | Dễ bị abuse hoặc DDoS nhẹ |
| Không rotate JWT secret | Không tuân thủ zero-trust, risk nếu secret bị leak |
| Lưu token frontend bằng `localStorage` | Dễ bị XSS tấn công |

## 📎 Tài liệu liên quan

- Secrets: [ADR-003](./adr-003-secrets.md)
- IaC Terraform Strategy: [ADR-002](./adr-002-iac.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)

---
> “Security không phải là một module – mà là mindset toàn hệ thống.”
