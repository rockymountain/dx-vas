---
id: adr-015-deployment-strategy
title: ADR-015 - Chiến lược Triển khai cho hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-05-22
tags: [deployment, strategy, cloud-run, ci-cd, dx-vas]
---

# ADR-015: Chiến lược Triển khai (Deployment Strategy)

## Bối cảnh

Hệ thống dx-vas cần hỗ trợ vận hành ổn định cho nhiều trường thành viên (tenant), với các yêu cầu:

- Tách biệt môi trường của từng tenant để đảm bảo bảo mật và tính ổn định
- Dễ dàng mở rộng khi có tenant mới
- Quản lý deployment theo cụm dịch vụ riêng biệt (core / tenant)
- Đảm bảo khả năng tự động hóa CI/CD, rollback, và giám sát theo tenant

## Quyết định

### 1. Kiến trúc triển khai tổng thể

Hệ thống được chia làm 2 nhóm triển khai chính:

| Nhóm | Nội dung | Môi trường |
|------|----------|------------|
| **Shared Core Services** | Gateway, Auth Master, User Master, Notification Master, Redis, Pub/Sub | `dx-vas-core` |
| **Tenant Stack (per tenant)** | Frontend apps, Sub Auth, Sub User, CRM/SIS/LMS Adapters, Sub Notification | `dx-vas-tenant-abc`, `dx-vas-tenant-xyz`, ... |

Mỗi tenant có thể chạy độc lập trên một GCP project riêng → dễ quản lý billing, tài nguyên, bảo mật.

### 2. Mô hình project trên Google Cloud

| Project | Vai trò |
|---------|---------|
| `dx-vas-core` | Dịch vụ chung (Gateway, Auth/User Master, Redis, Pub/Sub) |
| `dx-vas-tenant-[name]` | Stack riêng của từng tenant |
| `dx-vas-monitoring` | Giám sát, alerting, log tập trung |
| `dx-vas-data` | Cloud SQL, BigQuery, GCS dùng chung hoặc chia theo namespace |

### 3. Tên miền và định danh dịch vụ

- Mỗi frontend tenant có domain riêng:  
  `admin.abcschool.edu.vn`, `portal.xyzschool.edu.vn`
- Mỗi dịch vụ backend được đặt theo chuẩn:  
  `auth-service-master`, `user-service-master`, `auth-service-[tenant]`, `user-service-[tenant]`, ...
- API Gateway định tuyến theo `tenant_id` hoặc domain/subdomain

### 4. Tách biệt deployment & pipeline

- CI/CD được chia theo module:
  - Core: `gateway/`, `auth-master/`, `user-master/`, ...
  - Tenant: `tenant-[name]/auth/`, `tenant-[name]/frontend/`, ...
- Mỗi tenant có thể deploy riêng không ảnh hưởng tenant khác
- Superadmin Webapp có thể khởi tạo tenant mới bằng script (`terraform + deploy`)

### 5. Giám sát & quản lý môi trường

- Mỗi tenant có log, alert riêng biệt (theo project)
- Có khả năng collect toàn bộ log về Stackdriver chung (theo `tenant_id`)
- Hỗ trợ môi trường `dev`, `staging`, `prod` riêng biệt per stack

## Hệ quả

✅ Ưu điểm:

- Mỗi trường có thể vận hành độc lập (từ xác thực → gửi thông báo)
- Dễ scale khi có thêm tenant
- Đảm bảo isolation về bảo mật, performance
- Dễ tổ chức CI/CD theo module rõ ràng

⚠️ Lưu ý:

- Cần chi phí triển khai thêm GCP project và stack per tenant
- Quản trị multi-project cần Terraform module chuẩn hóa
- Quy trình deploy tenant mới cần được chuẩn hóa kỹ càng (tự động càng tốt)

## Liên kết liên quan

- [`adr-019-project-layout.md`](./adr-019-project-layout.md)
- [`adr-006-auth-strategy.md`](./adr-006-auth-strategy.md)
- [`adr-007-rbac.md`](./adr-007-rbac.md)
- [`system-diagrams.md#5-sơ-đồ-triển-khai-hạ-tầng-deployment-diagram`](../architecture/system-diagrams.md#5-sơ-đồ-triển-khai-hạ-tầng-deployment-diagram)

---
> "Triển khai đúng không chỉ là push code – mà là kiểm soát rủi ro và tạo niềm tin."
