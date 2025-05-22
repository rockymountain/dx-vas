---
id: adr-003-secrets
title: ADR-003: Chiến lược Quản lý và Xoay vòng Secrets cho hệ thống dx_vas
status: accepted
author: DX VAS Security & DevOps Team
date: 2025-06-22
tags: [secrets, security, secret-rotation, dx_vas]
---

## 📌 Bối cảnh

Hệ thống **dx_vas** bao gồm nhiều dịch vụ triển khai trên GCP như:
- API Gateway
- Backend Service (CRM Adapter, LMS Proxy, Notification Service, v.v.)
- Frontend Webapp (Admin, Customer Portal)
- CI/CD Pipeline (GitHub Actions)

Các dịch vụ này cần sử dụng **secrets** để kết nối với:
- Database (Cloud SQL), Redis
- 3rd-party API (Zalo, Google OAuth, Firebase...)
- Hệ thống nội bộ (SSO, webhook)

Nếu secrets không được quản lý đúng cách (hardcoded, không xoay vòng), hệ thống có thể gặp rủi ro bảo mật nghiêm trọng.

## 🧠 Quyết định

**Áp dụng chiến lược quản lý và xoay vòng secrets tập trung bằng Google Secret Manager và GitHub Secrets, phân tách rõ giữa secrets runtime và CI/CD, định kỳ rotate và kiểm soát audit.**

## 🔐 Loại secrets và nơi lưu trữ

| Loại secrets | Ví dụ | Lưu tại |
|--------------|-------|----------|
| Runtime secrets | DB password, JWT key, OAuth client ID/secret | Google Secret Manager (per env) |
| CI/CD secrets | Terraform SA key, GitHub token, WIF config | GitHub Secrets |

## 📦 Cách sử dụng secrets

### 1. Runtime secrets trong Cloud Run
- Inject từ Google Secret Manager vào **biến môi trường** hoặc **file**
- Sử dụng Terraform để cấu hình binding IAM và mount
- Cloud Run tự động mount phiên bản mới nếu có update (optional warm reload logic)

### 2. CI/CD sử dụng secrets
- GitHub Actions đọc từ GitHub Secrets:
  - `WIF_PROVIDER`, `WIF_SERVICE_ACCOUNT`
  - `JWT_SIGNING_KEY`, `SLACK_WEBHOOK_URL`
- Secrets không bao giờ được ghi log hoặc in ra output CI

## 🔁 Chính sách xoay vòng (Rotation Policy)

| Loại secrets | Chu kỳ xoay | Ghi chú |
|--------------|-------------|--------|
| DB password | 90 ngày | Qua script tự động hoặc Cloud SQL IAM |
| JWT signing key | 30 ngày | Hỗ trợ multiple keys & forward compatibility |
| OAuth/Zalo key | 60–90 ngày hoặc khi revoke |
| GitHub token | 90 ngày hoặc khi bị rotate bởi GitHub |

- Xoay vòng được thực hiện qua:
  - `gcloud secrets versions add`
  - Terraform apply mới (chuyển secret version)
  - Gửi deploy mới để mount lại secrets vào Cloud Run

### ➕ Tự động hoá
- Sử dụng Cloud Scheduler + Pub/Sub để xoay định kỳ
- Có thể viết bot CI tự động rotate một số token OAuth expiring

## 🔎 Kiểm soát & Audit

- Audit access qua **Cloud Audit Logs** và `access logs` của GitHub
- Gắn tag secret theo: `env`, `application`, `rotation_policy`
- CI/CD scan check trong pre-commit/CI:
  - Không cho phép push hardcoded secret (dùng `gitleaks`, `truffleHog`)

## ✅ Lợi ích

- Bảo mật tập trung, dễ kiểm soát quyền truy cập theo môi trường
- Dễ dàng thay thế/revoke nếu rò rỉ
- Tăng khả năng tuân thủ các tiêu chuẩn (OWASP, Google Cloud best practices)
- Gắn chặt quy trình security với CI/CD pipeline

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Quên rotate secrets | Alert sau 90 ngày qua CI hoặc secret label + audit log |
| Secret bị ghi log | Dùng middleware log masking, CI block output chứa `TOKEN`, `SECRET` |
| Secret bị revoke nhưng chưa update | Deploy mỗi lần rotate, hỗ trợ rollback version cũ trong Secret Manager |

## 🔄 Các lựa chọn đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| `.env` file trong Git repo | Rò rỉ bảo mật, không kiểm soát phân quyền |
| Lưu tất cả secrets trong GitHub Secrets | Không phù hợp cho runtime GCP, không hỗ trợ IAM riêng theo env |
| Không rotate | Không tuân thủ zero-trust, rủi ro khi credential bị lộ |

## 📎 Tài liệu liên quan

- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)
- IaC Terraform Strategy: [ADR-002](./adr-002-iac.md)

---
> “Không có secret nào nên tồn tại mãi mãi – rotate là cách bạn bảo vệ hệ thống khi bạn quên rằng đã từng để lộ nó.”
