---
id: adr-005-env-config
title: ADR-005 - Chiến lược Cấu hình đa môi trường (Environment Configuration) cho hệ thống dx-vas
status: accepted
author: DX VAS DevOps Team
date: 2025-05-22
tags: [configuration, environment, dotenv, ci-cd, secrets, dx-vas]
---

## 📌 Bối cảnh

Hệ thống **dx-vas** vận hành trên nhiều môi trường:

* `dev`: phát triển tính năng, kiểm thử local
* `staging`: kiểm thử tích hợp, demo nội bộ
* `production`: môi trường chính thức với người dùng thực
* `sandbox`: thử nghiệm độc lập, testing API công khai

Mỗi môi trường yêu cầu cấu hình riêng biệt cho:

* Biến môi trường: URL backend, CORS, log level, feature flag
* Secrets: DB password, API token, JWT key
* Quản lý trong CI/CD và Terraform rõ ràng

## 🧠 Quyết định

**Áp dụng chiến lược cấu hình theo môi trường sử dụng file `.env`, biến hệ thống, và secrets từ Secret Manager/GitHub Secrets, đảm bảo tách biệt, dễ kiểm soát và tự động hoá.**

## 📁 Cấu trúc đề xuất

Chiến lược quản lý file cấu hình được áp dụng cho **từng repository của mỗi service/ứng dụng con**:

**Phương án: Sử dụng thư mục con `config/` bên trong mỗi service** (được ưu tiên để giữ thư mục gốc sạch sẽ và dễ mở rộng khi số lượng file cấu hình tăng):

```bash
# Ví dụ, trong repo của API Gateway (api_gateway/):
config/
  ├── base.env       # Biến chung cho service (không nhạy cảm)
  ├── dev.env        # Override cho dev (không commit nếu chứa secret)
  ├── staging.env    # Override cho staging (không commit nếu chứa secret)
  └── production.env # Override cho production (không commit nếu chứa secret)
.env.example          # Template ở gốc
```

* `base.env`: chứa các biến cấu hình chung cho **service đó**, không phải toàn hệ thống dx-vas
* Các file `.env.{ENVIRONMENT}` hoặc `{environment}.env`: chứa các giá trị override cụ thể cho từng môi trường
* Dùng với `python-dotenv`, `pydantic-settings`, hoặc `dotenv` của NodeJS
* Không commit `.env` chứa secrets – sử dụng placeholder hoặc `.example.env`

## 📦 Biến môi trường quan trọng

| Biến                | Ý nghĩa                                        |
| ------------------- | ---------------------------------------------- |
| `ENV`               | `dev`, `staging`, `production`, `sandbox`      |
| `BACKEND_URL_CRM`   | endpoint tích hợp CRM                          |
| `JWT_SECRET`        | key ký access token (inject từ Secret Manager) |
| `LOG_LEVEL`         | `debug` / `info` / `warning`                   |
| `FEATURE_X_ENABLED` | `true/false` – bật feature theo env            |

## 🔧 Cách nạp cấu hình

Thứ tự ưu tiên khi load config:

1. Biến môi trường từ hệ thống (`os.environ`) hoặc CI/CD inject
2. File `config/{ENV}.env` nếu chạy local
3. Mặc định trong mã nguồn (nếu không phải secret)

> Khi lập trình viên chạy local từ nhánh `feature/*` hoặc `dev`, có thể đặt `ENV=dev` để sử dụng cấu hình trong `config/dev.env`. Nếu muốn mô phỏng gần với môi trường staging, có thể đặt `ENV=staging` để kiểm thử với cấu hình staging.

Framework đề xuất:

* Python: `pydantic.BaseSettings`
* NodeJS: `dotenv` + `envsafe`
* Frontend: Next.js dùng `.env.local`, `.env.production`

## 🔐 Secrets tách biệt khỏi `.env`

* Secrets nhạy cảm không lưu trong `.env` mà được mount từ:

  * GCP Secret Manager (runtime)
  * GitHub Secrets (CI/CD)
  * Terraform `TF_VAR_` inject

## 🛠 Trong CI/CD

* CI xác định giá trị cho biến `ENV` (để load đúng file cấu hình) và môi trường triển khai dựa trên nhánh:

  * Khi CI chạy trên nhánh `dev`:

    * Sẽ deploy lên môi trường **Staging**.
    * Biến `ENV` được thiết lập là `staging` để load `config/staging.env`.
  * Khi CI chạy trên nhánh `main`:

    * Sẽ deploy lên môi trường **Production** (sau khi có phê duyệt).
    * Biến `ENV` được thiết lập là `production` để load `config/production.env`.
  * Khi chạy local trên máy lập trình viên (ví dụ từ nhánh `feature/*` hoặc `dev`):

    * Biến `ENV` có thể được đặt là `dev` để load `config/dev.env` phục vụ phát triển và kiểm thử cục bộ.

* CI sẽ inject các biến cấu hình vào môi trường Cloud Run dựa trên giá trị `ENV`, ưu tiên secrets từ Secret Manager và các biến môi trường từ pipeline.

## ✅ Lợi ích

* Cấu hình rõ ràng theo môi trường
* Dễ debug và chạy local giống production
* Không rò rỉ secrets nhạy cảm
* Dễ tích hợp vào Terraform và Cloud Run deployment

## ❌ Rủi ro & Giải pháp

| Rủi ro                             | Giải pháp                                          |
| ---------------------------------- | -------------------------------------------------- |
| Commit nhầm file `.env`            | Dùng `.gitignore`, CI scan block file              |
| Thiếu key cấu hình gây lỗi runtime | Script validate schema + fallback mặc định an toàn |
| Config bị leak qua log CI          | Mask biến môi trường chứa `SECRET`, `TOKEN`, `KEY` |

## 🔄 Các phương án đã loại bỏ

| Phương án                        | Lý do không chọn                        |
| -------------------------------- | --------------------------------------- |
| Hardcode config trong code       | Khó thay đổi, không phù hợp CI/CD       |
| Dùng chung `.env` mọi môi trường | Dễ ghi đè sai env, thiếu tách biệt      |
| Config toàn bộ qua Terraform     | Không phù hợp frontend, khó debug local |

## 📎 Tài liệu liên quan

* Secrets: [ADR-003](./adr-003-secrets.md)
* IaC Terraform Strategy: [ADR-002](./adr-002-iac.md)
* CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)

---

> “Một hệ thống chạy tốt trên production – là hệ thống có cấu hình rõ ràng từ đầu.”
