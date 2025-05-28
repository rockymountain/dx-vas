---
id: adr-001-ci-cd
title: ADR-001 - Chiến lược CI/CD chung cho toàn hệ thống dx-vas
status: accepted
author: DX VAS DevOps Team
date: 2025-05-22
tags: [ci, cd, github-actions, cloud-run, dx-vas]
---

## 📌 Bối cảnh

Dự án **Chuyển đổi số VAS (dx-vas)** bao gồm nhiều thành phần:
- API Gateway
- Các backend service tích hợp (CRM Adapter, SIS Adapter, Notification Service, LMS Proxy...)
- Frontend (Admin Webapp, Customer Portal)

Các thành phần này đều được triển khai trên nền tảng **Google Cloud Run**, yêu cầu CI/CD:
- Tự động build, test, scan bảo mật trước khi deploy
- Phân biệt rõ môi trường `dev`, `staging`, `production`
- Hạn chế lỗi thủ công khi deploy thủ công hoặc thiếu test

## 🧠 Quyết định

**Áp dụng chiến lược CI/CD tập trung sử dụng GitHub Actions, deploy các service dx-vas lên Cloud Run, phân tách branch theo môi trường và enforce kiểm tra bảo mật/tài liệu trước khi release.**

## 🛠 Chi tiết thiết kế

### 1. Branching Strategy

| Branch | Môi trường | Deploy tự động |
|--------|------------|-----------------|
| `dev` | Staging | ✅ Có |
| `main` | Production | ✅ Có (sau approval) |
| `feature/*` | Dev local/test | ❌ Không |

### 2. Workflow CI/CD chuẩn

```yaml
on:
  push:
    branches: [dev, main]

jobs:
  ci:
    steps:
      - Checkout, setup Python/NodeJS
      - Cài đặt phụ thuộc:
        - Nếu dùng `requirements.txt`: `pip install -r requirements.txt`
        - Nếu dùng Poetry: `poetry install`
        - Nếu dùng Pipenv: `pipenv install --dev`
      - Chạy lint: `black`, `flake8`, `eslint`, `prettier`
      - Chạy test + coverage: `pytest`, `jest`
      - Chạy scan bảo mật: `bandit`, `safety`, `trivy`

  cd:
    needs: ci
    steps:
      - Build Docker image, tag theo `env`, `git_sha`
      - Deploy Cloud Run bằng `gcloud deploy` hoặc `google-github-actions`
      - Đặt `min-instances`, `concurrency`, `env var`, secret (qua Secret Manager)
```

### 3. Secrets

| Loại | Lưu ở đâu |
|------|-----------|
| CI/CD secrets | GitHub Secrets |
| Runtime secrets | Google Secret Manager |

### 4. Rule bắt buộc

- ✅ Pull Request phải được review
- ✅ Chạy CI tự động trước merge
- ✅ Lỗi lint/test/security đều chặn merge
- ✅ Production deploy cần manual approval hoặc protected branch
- ✅ **Mọi thay đổi hạ tầng (Terraform) phải chạy `terraform plan` qua CI và được review trước khi `apply`**

### 5. Template thư mục CI trong mỗi repo

```
.github/workflows/
  ci.yml
  deploy.yml
  lint.yml (optional)
```

## ✅ Lợi ích

- Tự động hoá quy trình build/deploy toàn hệ thống
- Phát hiện lỗi sớm ngay khi commit
- Thống nhất convention giữa các team backend/frontend
- Giảm rủi ro bảo mật và lỗi triển khai production

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Quên chạy test trước khi merge | CI tự động enforce |
| Lộ secrets qua log | Bật masking, không echo giá trị nhạy cảm |
| Deploy nhầm môi trường | Tách rõ `dev`, `main`, yêu cầu approval cho `main` |

## 🔄 Các lựa chọn đã loại bỏ

- **Deploy thủ công**: dễ sai, không rollback
- **Jenkins/GitLab CI**: không đồng bộ với GitHub codebase

## 📎 Tài liệu liên quan

- Terraform module: `infra/modules/cloud_run_service`
- IaC Terraform Strategy: [ADR-002](./adr-002-iac.md)

---
> "Mỗi lần commit là một lời hứa – CI/CD giúp đảm bảo lời hứa đó được thực hiện."
