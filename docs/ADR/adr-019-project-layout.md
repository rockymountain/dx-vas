---
id: adr-019-project-layout
title: ADR-019: Chiến lược tổ chức GCP Project, Network & Quyền hạn cho hệ thống dx_vas
status: draft
author: DX VAS Platform Team
date: 2025-06-22
tags: [iac, gcp, terraform, networking, multi-project, dx_vas]
---

## 📌 Bối cảnh

Hệ thống **dx_vas** đang phát triển với nhiều dịch vụ và môi trường vận hành:
- `dev`, `staging`, `production` có thể cần tách biệt
- Một số dịch vụ có thể chia sẻ VPC, DNS, Redis hoặc Cloud SQL
- Nhiều team cùng tham gia triển khai và cần ranh giới IAM rõ ràng

Do đó cần chuẩn hoá việc:
- Tổ chức GCP project và naming convention
- Chia tách tài nguyên theo môi trường
- Phân quyền triển khai (CI/CD) và vận hành
- Kết nối mạng chéo project (Shared VPC hoặc Private Services)

---

## 🧠 Quyết định

**Áp dụng mô hình multi-project tách biệt theo môi trường (`dev`, `staging`, `prod`) và theo từng module chức năng (nếu cần). Thiết lập Shared VPC nếu cần, phân quyền IAM theo team và CI/CD theo từng project cụ thể.**

---

## 🧭 Cấu trúc đề xuất

### 🌐 Project layout

| Project ID | Môi trường | Mục đích |
|------------|------------|----------|
| dx-vas-dev | development | Dev, QA nội bộ, test CI/CD |
| dx-vas-staging | staging | Kiểm thử tích hợp trước khi production |
| dx-vas-prod | production | Vận hành thực tế, SLA cao |

### 🧱 Chia tách theo module (tuỳ chọn)
- Nếu mở rộng lớn, có thể tách `dx-vas-network`, `dx-vas-logging`, `dx-vas-data` làm module riêng cho hạ tầng chia sẻ

### 📡 Network & DNS
- Dùng **Shared VPC** từ `dx-vas-network`
- Cloud DNS zone chung (`internal.dxvas.local.`)
- Firewall, NAT config từ module trung tâm

### 🔐 IAM & CI/CD
- Mỗi project có nhóm quyền riêng:
  - `ci-deploy-dev@` → chỉ deploy dev
  - `platform-admin-prod@` → chỉ vận hành prod
- Terraform tách `backend.tf` và `provider.tf` theo project + environment

---

## 🛠 Terraform Structure

```bash
infrastructure/
├── modules/
│   ├── network/
│   ├── gke/
│   └── redis/
├── environments/
│   ├── dev/
│   ├── staging/
│   └── prod/
└── shared/
    ├── project.tf
    └── dns.tf
```

> Ghi chú: cập nhật thống nhất thư mục `environments/` trong tất cả ADR liên quan như `adr-002-iac.md`.

---

## ✅ Lợi ích

- Tách biệt rõ môi trường, dễ kiểm soát và audit
- Cho phép team phát triển thử nghiệm độc lập
- Dễ áp dụng chính sách cost, alert và IAM riêng theo env

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| IAM phức tạp & dễ sai | Chuẩn hoá Terraform module IAM, dùng service account cố định |
| Chi phí chia nhỏ project khó theo dõi | Gắn label `dx_vas_service`, `env`, `team` để cost attribution |
| Truy cập chéo project lỗi | Xác định rõ source project ↔ target service, áp dụng VPC peering / PSC |

---

## 📎 Tài liệu liên quan

- IaC Strategy: [ADR-002](./adr-002-iac.md)
- CI/CD: [ADR-001](./adr-001-ci-cd.md)
- Observability: [ADR-005](./adr-005-observability.md)
- Cost Observability: [ADR-020](./adr-020-cost-observability.md)

---
> “Cấu trúc project tốt là nền tảng cho scale, bảo mật và kiểm soát chi phí.”
