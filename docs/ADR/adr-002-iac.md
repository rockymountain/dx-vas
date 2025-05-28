---
id: adr-002-iac
title: ADR-002 - Chiến lược Hạ tầng dưới dạng Mã nguồn (Infrastructure as Code) cho hệ thống dx-vas
status: accepted
author: DX VAS DevOps Team
date: 2025-05-22
tags: [iac, terraform, infrastructure, dx-vas]
---

## 📌 Bối cảnh

Hệ thống **dx-vas** bao gồm nhiều thành phần triển khai trên Google Cloud Platform (GCP):

* API Gateway
* Backend service (LMS Adapter, CRM Adapter, Notification Service)
* Frontend (Admin Webapp, Customer Portal)
* Database, Redis, Secret Manager, IAM, Monitoring

Việc triển khai và cấu hình các tài nguyên hạ tầng cần được:

* Tái sử dụng và lặp lại được (reproducible)
* Theo dõi thay đổi qua Git (version control)
* Tự động hóa, kiểm thử được trong CI/CD
* Chia tách theo môi trường (`dev`, `staging`, `production`, `sandbox`)

## 🧠 Quyết định

**Áp dụng Terraform làm công cụ chính quản lý hạ tầng dx-vas, với mô hình tách module + môi trường, sử dụng thư mục `envs/` để chứa cấu hình theo môi trường, và CI pipeline kiểm soát apply, state và secrets an toàn.**

## 🧱 Cấu trúc đề xuất

```bash
dx-vas/infrastructure/
├── modules/
│   ├── cloud_run_service/
│   ├── cloud_sql_instance/
│   ├── redis_instance/
│   ├── iam_roles/
│   ├── monitoring/
│   └── storage/
│
├── envs/
│   ├── dev/
│   ├── staging/
│   ├── production/
│   └── sandbox/
│
├── shared/
│   ├── dx-vas-network/
│   │   ├── main.tf           # Shared VPC, subnet, firewall, NAT
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── dx-vas-logging/
│   │   ├── main.tf           # Logging sinks, audit config, BigQuery export
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── dx-vas-data/
│   │   ├── main.tf           # Redis, Cloud SQL, BigQuery dataset
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── dns.tf                # DNS zones chung
│   └── README.md
│
├── backend.tf     # cấu hình state backend
└── README.md
```

> 🔁 Mỗi thư mục trong `envs/` gồm `main.tf`, `variables.tf`, `outputs.tf`, và `*.tfvars`. Thư mục `shared/` quản lý hạ tầng chia sẻ được sử dụng bởi nhiều môi trường hoặc service, nhất quán với ADR-019.

## ⚙️ Nguyên tắc sử dụng

* **Tách module dùng chung**: các service chỉ cần gọi lại module với biến riêng
* **State file lưu GCS bucket** (per env), có versioning và encryption
* **Secrets** không hard-code, được inject qua CI/CD hoặc GCP Secret Manager
* **Terraform workspace** được dùng thêm nếu môi trường có nhiều biến tạm

## 🔐 Quản lý secrets

| Loại secret           | Lưu ở đâu          | Truy cập từ        |
| --------------------- | ------------------ | ------------------ |
| Runtime token, DB URL | GCP Secret Manager | Cloud Run Service  |
| CI/CD secret          | GitHub Secrets     | Terraform apply CI |

## 🔄 Quy trình CI/CD

* Chạy `terraform validate`, `plan` → tạo PR
* Artifact `.tfplan` được lưu để review
* `apply` tự động cho `dev`, cần approval với `production`
* Lint Terraform với `tflint`, kiểm tra drift với `infracost`, `terraform-docs`

## 📌 Chính sách an toàn

* ✅ Mọi thay đổi hạ tầng phải được review qua Pull Request
* ✅ Không deploy thẳng từ máy local lên production
* ✅ Có policy check và scan drift trước apply (sử dụng `OPA` hoặc `checkov` nếu cần)
* ✅ CI sẽ báo lỗi nếu thiếu tag/label quan trọng (`env`, `owner`, `component`...)

## ✅ Lợi ích

* Tái sử dụng được hạ tầng cho nhiều service
* Quản lý rõ ràng theo môi trường (cấu hình, chi phí, biến runtime)
* Kiểm soát, rollback, và kiểm toán hạ tầng như code
* Kết nối CI/CD an toàn, tránh deploy sai cấu hình

## ❌ Rủi ro & Giải pháp

| Rủi ro                                    | Giải pháp                                          |
| ----------------------------------------- | -------------------------------------------------- |
| Quên cập nhật `tfvars` cho môi trường mới | Có template `.example.tfvars` và validate trong CI |
| State bị khoá hoặc mất đồng bộ            | Enable state lock, versioning trong GCS            |
| Drift do chỉnh tay trong Console          | `terraform plan` + check drift + policy check CI   |

## 🔄 Phương án khác đã loại bỏ

| Phương án            | Lý do không chọn                                        |
| -------------------- | ------------------------------------------------------- |
| GCP Console thủ công | Không version control, dễ sai lệch giữa env             |
| Deployment Manager   | Ít phổ biến, hạn chế cộng đồng và tài liệu              |
| Pulumi               | Mạnh nhưng yêu cầu học SDK + không đồng nhất trong team |

## 📎 Tài liệu liên quan

* CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)
* Secret Management: [ADR-003](./adr-003-secrets.md)
* Project Layout: [ADR-019](./adr-019-project-layout.md)

---

> “IaC không chỉ là cách viết hạ tầng – nó là cách làm chủ sự phát triển của toàn bộ hệ thống.”
