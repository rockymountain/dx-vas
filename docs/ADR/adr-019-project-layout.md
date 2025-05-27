---
id: adr-019-project-layout
title: ADR-019: Chiến lược tổ chức GCP Project, Network & Quyền hạn cho hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [iac, gcp, terraform, networking, multi-project, dx-vas]
---

# ADR-019: Chiến lược chia tách project GCP (Project Layout Strategy)

## Bối cảnh

Hệ thống dx-vas triển khai trên Google Cloud Platform (GCP), với yêu cầu:

- Phân chia rõ ràng tài nguyên giữa các khối chức năng (core vs tenant)
- Hỗ trợ tách chi phí theo tenant
- Dễ mở rộng khi có thêm trường mới
- Hỗ trợ triển khai, giám sát và bảo mật theo từng đơn vị độc lập

## Quyết định

### 1. Phân chia project theo nhóm chức năng

Hệ thống được chia thành các GCP project như sau:

| Project | Vai trò |
|---------|---------|
| `dx-vas-core` | Dịch vụ chung toàn hệ thống: Gateway, Auth Master, User Master, Notification Master, Redis, Pub/Sub |
| `dx-vas-tenant-abc`, `dx-vas-tenant-xyz`, ... | Mỗi trường (tenant) có project riêng chứa: Frontend App, Sub Auth, Sub User, CRM/SIS/LMS Adapter, Sub Notification |
| `dx-vas-monitoring` | Log tập trung, Alerting, Metrics (Stackdriver / Cloud Monitoring) |
| `dx-vas-data` | Cloud SQL, BigQuery, GCS phục vụ dữ liệu chia sẻ hoặc tích hợp BI |

### 2. Ưu điểm của mô hình này

- **Isolation mạnh:** mỗi tenant cách ly về network, tài nguyên, log, secret
- **Tách billing:** chi phí theo tenant, phục vụ báo cáo nội bộ hoặc khách hàng
- **Tự động hóa linh hoạt:** mỗi project có thể được khởi tạo từ module Terraform riêng
- **Phân quyền IAM rõ ràng:** team DevOps từng trường chỉ cần quyền trên project tương ứng

### 3. Nguyên tắc định danh dịch vụ

- Tên dịch vụ theo chuẩn:  
  - `auth-service-master`, `user-service-master`, `notification-master`
  - `auth-service-[tenant]`, `user-service-[tenant]`, `frontend-[tenant]`, ...
- Subdomain theo tenant:  
  - `admin.abcschool.edu.vn`, `portal.xyzschool.edu.vn`

### 4. Networking & truy cập nội bộ

- Các service nội bộ giao tiếp qua API Gateway (triển khai tại `dx-vas-core`)
- Gateway định tuyến theo `tenant_id`, hoặc host/domain tùy context
- Nếu cần giao tiếp nội bộ (master → sub), sử dụng service token hoặc mTLS + header `X-Internal-Call`
- Sub Service chỉ nhận request từ API Gateway (qua VPC connector hoặc ingress restriction)

### 5. Terraform & Tự động hóa

- Mỗi loại project (core, tenant, monitoring) có module Terraform riêng
- Tạo tenant mới = tạo project + stack = `terraform apply -var="tenant=abc"`
- Biến môi trường, secrets, domain được cấu hình tự động qua CI/CD

## Hệ quả

✅ Ưu điểm:

- Tách bạch rõ ràng, dễ quản trị và bảo mật
- Triển khai song song, không ảnh hưởng giữa các tenant
- Scale lên hàng chục tenant vẫn giữ được độ rõ ràng
- Phù hợp mô hình team DevOps chia theo trường

⚠️ Lưu ý:

- Tăng số lượng project cần quản lý → cần có công cụ tổng thể (Cloud Resource Manager, Folder Policy)
- IAM phức tạp hơn nếu không có quy ước đặt tên & phân quyền chuẩn

## Liên kết liên quan

- [`adr-015-deployment-strategy.md`](./adr-015-deployment-strategy.md)
- [`adr-004-security.md`](./adr-004-security.md)
- [`README.md#8-hạ-tầng-triển-khai`](../README.md#8-hạ-tầng-triển-khai)
- [`system-diagrams.md#5-sơ-đồ-triển-khai-hạ-tầng`](../architecture/system-diagrams.md#5-sơ-đồ-triển-khai-hạ-tầng-deployment-diagram)

---

> “Kiến trúc tốt bắt đầu từ một nền móng project rõ ràng.”
