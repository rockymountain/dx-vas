---

id: adr-019-project-layout
title: ADR-019: Chiến lược tổ chức GCP Project, Network & Quyền hạn cho hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [iac, gcp, terraform, networking, multi-project, dx-vas]
---

## 📌 Bối cảnh

Hệ thống **dx-vas** đang phát triển với nhiều dịch vụ microservice, triển khai qua nhiều môi trường (`dev`, `staging`, `production`) và cần tách biệt rõ ràng để đảm bảo:

* An toàn khi thử nghiệm tính năng mới
* Kiểm soát chi phí và quyền truy cập theo môi trường
* Quản trị hạ tầng phức tạp với Shared VPC, CI/CD, logging, observability

Do đó, cần một chiến lược tổ chức GCP project và network rõ ràng, hỗ trợ IaC, RBAC, giám sát và scale tốt.

---

## 🧠 Quyết định

**Áp dụng mô hình multi-project theo môi trường (`dev`, `staging`, `prod`) và module hạ tầng dùng chung, với phân quyền IAM tách biệt, Shared VPC, và tích hợp CI/CD + cost tracking chuẩn hóa.**

---

## 🧭 Cấu trúc Project đề xuất

### 🌍 GCP Project per environment

| Project ID       | Môi trường | Mục đích                                |
| ---------------- | ---------- | --------------------------------------- |
| `dx-vas-dev`     | dev        | Local test, staging feature, CI/CD      |
| `dx-vas-staging` | staging    | Tích hợp liên dịch vụ, QA trước release |
| `dx-vas-prod`    | production | Vận hành chính thức, SLA cao            |

> Mỗi project có billing riêng, secret riêng, IAM policy riêng.

### 🧱 Module hạ tầng chung

| Project ID       | Mục đích                           |
| ---------------- | ---------------------------------- |
| `dx-vas-network` | Shared VPC, Cloud DNS, NAT gateway |
| `dx-vas-logging` | Centralized logging + audit        |
| `dx-vas-data`    | Shared Redis, Cloud SQL, BigQuery  |

Các project này được dùng chung bởi môi trường `dev/staging/prod` qua Shared VPC hoặc VPC Peering.

---

## 🔐 IAM & CI/CD

* CI/CD phân tách rõ quyền deploy:

  * `ci-deploy-dev@` → chỉ có role deploy vào `dx-vas-dev`
  * `ci-deploy-staging@` → chỉ staging
  * `ci-deploy-prod@` → cần phê duyệt theo \[ADR-018]

* Terraform tách `provider.tf` theo `project_id`, biến môi trường xác định `env`

* Dùng Workload Identity Federation để tránh dùng key tĩnh

---

## 📡 Mạng & DNS

* Sử dụng **Shared VPC** từ `dx-vas-network`
* VPC Subnet theo env (`10.10.0.0/16`, `10.20.0.0/16`, ...)
* DNS zone nội bộ: `internal.dxvas.local.`
* Dùng Private Google Access để không cần IP public

---

## 🛠 Terraform Layout

```bash
infrastructure/
├── modules/
│   ├── cloud_run_service/
│   ├── redis_instance/
│   ├── cloud_sql_instance/
│   └── iam_roles/
├── envs/
│   ├── dev/
│   ├── staging/
│   └── prod/
├── shared/
│   ├── network/
│   ├── dns.tf
│   └── logging.tf
└── backend.tf / README.md
```

> Tất cả `envs/` đều chứa `main.tf`, `variables.tf`, `outputs.tf`, và `.tfvars`

---

## 🏷️ Tag & Cost Tracking

* Gắn label cho mọi tài nguyên:

  * `dx-vas_service`, `env`, `owner`, `critical`
* Dùng label cho Cloud Billing + BigQuery export (ADR-020)
* Dashboard chi phí chia theo project, env, module

---

## ✅ Lợi ích

* Phân tách rõ ràng giữa các môi trường giúp deploy an toàn hơn
* Dễ kiểm soát quyền, truy cập, và trace chi phí
* Hạ tầng mở rộng được theo module: network, log, storage
* Phù hợp với CI/CD, Terraform, Shared VPC

---

## ❌ Rủi ro & Giải pháp

| Rủi ro                      | Giải pháp                                            |
| --------------------------- | ---------------------------------------------------- |
| IAM phức tạp                | Sử dụng module Terraform + group IAM rõ ràng         |
| Gọi chéo project không được | Kiểm soát `Service Account`, VPC Peering, Shared VPC |
| Cost phân tán khó theo dõi  | Gắn label chuẩn hóa + dashboard BI từ BigQuery       |

---

## 📎 Tài liệu liên quan

* IaC Strategy: [ADR-002](./adr-002-iac.md)
* CI/CD: [ADR-001](./adr-001-ci-cd.md)
* Cost Observability: [ADR-020](./adr-020-cost-observability.md)
* Env Deploy Boundary: [ADR-017](./adr-017-env-deploy-boundary.md)
* Release Approval: [ADR-018](./adr-018-release-approval-policy.md)

---

> “Kiến trúc tốt bắt đầu từ một nền móng project rõ ràng.”
