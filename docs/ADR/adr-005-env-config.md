---
id: adr-005-env-config
title: ADR-005: Chiến lược Cấu hình đa môi trường (Environment Configuration) cho hệ thống dx_vas
status: accepted
author: DX VAS DevOps Team
date: 2025-06-22
tags: [configuration, environment, dotenv, ci-cd, secrets, dx_vas]
---

## 📌 Bối cảnh

Hệ thống **dx_vas** vận hành trên nhiều môi trường:
- `dev`: phát triển tính năng, kiểm thử local
- `staging`: kiểm thử tích hợp, demo nội bộ
- `production`: môi trường chính thức với người dùng thực
- `sandbox`: thử nghiệm độc lập, testing API công khai

Mỗi môi trường yêu cầu cấu hình riêng biệt cho:
- Biến môi trường: URL backend, CORS, log level, feature flag
- Secrets: DB password, API token, JWT key
- Quản lý trong CI/CD và Terraform rõ ràng

## 🧠 Quyết định

**Áp dụng chiến lược cấu hình theo môi trường sử dụng file `.env`, biến hệ thống, và secrets từ Secret Manager/GitHub Secrets, đảm bảo tách biệt, dễ kiểm soát và tự động hoá.**

## 📁 Cấu trúc đề xuất

Chiến lược quản lý file cấu hình được áp dụng cho **từng repository của mỗi service/ứng dụng con**. Có hai phương án chính cho mỗi service:

**Phương án A: Các file `.env` theo môi trường ở thư mục gốc của service:**
```bash
# Ví dụ, trong repo của API Gateway (api_gateway/):
.env.example        # Template các biến cần thiết
.env.development    # Cấu hình cho môi trường development local
.env.staging        # Cấu hình cho môi trường staging (nếu load từ file khi dev)
.env.production     # Cấu hình cho môi trường production (nếu load từ file khi dev)
# Các file .env.* này (trừ .env.example) KHÔNG được commit vào Git.
```

**Phương án B: Sử dụng thư mục con `config/` bên trong mỗi service:**
```bash
# Ví dụ, trong repo của API Gateway (api_gateway/):
config/
  ├── base.env          # Biến chung cho service (không nhạy cảm)
  ├── development.env   # Override cho dev (không commit nếu chứa secret)
  ├── staging.env       # Override cho staging (không commit nếu chứa secret)
  └── production.env    # Override cho production (không commit nếu chứa secret)
.env.example            # Template ở gốc
```

*Lựa chọn giữa Phương án A và B tùy thuộc vào quy ước của từng đội phát triển service, nhưng cần đảm bảo nguyên tắc tách biệt theo môi trường và không commit secrets.*

- `base.env`: chứa các biến cấu hình chung cho **service đó**, không phải toàn hệ thống dx_vas
- Các file `.env.{ENVIRONMENT}` hoặc `{environment}.env`: chứa các giá trị override cụ thể cho từng môi trường
- Dùng với `python-dotenv`, `pydantic-settings`, hoặc `dotenv` của NodeJS
- Không commit `.env` chứa secrets – sử dụng placeholder hoặc `.example.env`

## 📦 Biến môi trường quan trọng

| Biến | Ý nghĩa |
|------|--------|
| `ENV` | `dev`, `staging`, `production`, `sandbox` |
| `BACKEND_URL_CRM` | endpoint tích hợp CRM |
| `JWT_SECRET` | key ký access token (inject từ Secret Manager) |
| `LOG_LEVEL` | `debug` / `info` / `warning` |
| `FEATURE_X_ENABLED` | `true/false` – bật feature theo env |

## 🔧 Cách nạp cấu hình

Thứ tự ưu tiên khi load config:
1. Biến môi trường từ hệ thống (`os.environ`) hoặc CI/CD inject
2. File `.env.{ENV}` nếu chạy local
3. Mặc định trong mã nguồn (nếu không phải secret)

Framework đề xuất:
- Python: `pydantic.BaseSettings`
- NodeJS: `dotenv` + `envsafe`
- Frontend: Next.js dùng `.env.local`, `.env.production`

## 🔐 Secrets tách biệt khỏi `.env`

- Secrets nhạy cảm không lưu trong `.env` mà được mount từ:
  - GCP Secret Manager (runtime)
  - GitHub Secrets (CI/CD)
  - Terraform `TF_VAR_` inject

## 🛠 Trong CI/CD

- CI xác định môi trường dựa trên nhánh:
  - `dev` → `ENV=staging`
  - `main` → `ENV=production`
- Inject qua `env:` block hoặc `.env` file
- Kiểm tra config tồn tại đủ key bằng script validate

## ✅ Lợi ích

- Cấu hình rõ ràng theo môi trường
- Dễ debug và chạy local giống production
- Không rò rỉ secrets nhạy cảm
- Dễ tích hợp vào Terraform và Cloud Run deployment

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Commit nhầm file `.env` | Dùng `.gitignore`, CI scan block file |
| Thiếu key cấu hình gây lỗi runtime | Script validate schema + fallback mặc định an toàn |
| Config bị leak qua log CI | Mask biến môi trường chứa `SECRET`, `TOKEN`, `KEY` |

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Hardcode config trong code | Khó thay đổi, không phù hợp CI/CD |
| Dùng chung `.env` mọi môi trường | Dễ ghi đè sai env, thiếu tách biệt |
| Config toàn bộ qua Terraform | Không phù hợp frontend, khó debug local |

## 📎 Tài liệu liên quan

- Secrets: [ADR-003](./adr-003-secrets.md)
- IaC Terraform Strategy: [ADR-002](./adr-002-iac.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)

---
> “Một hệ thống chạy tốt trên production – là hệ thống có cấu hình rõ ràng từ đầu.”
