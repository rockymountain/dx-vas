---
id: adr-025-multi-tenant-versioning
title: ADR-025 - Chiến lược Phiên bản hóa Tenant Stack
status: accepted
author: DX VAS DevOps Team
date: 2025-05-22
tags: [multi-tenant, versioning, deployment, ci-cd, tenant-stack, release-management, backward-compatibility, staging, rollout-strategy]
---

# ADR-025: Chiến lược Phiên bản hóa Tenant Stack (Multi-Tenant Versioning Strategy)

## Bối cảnh

Hệ thống dx-vas triển khai theo mô hình multi-tenant với mỗi trường (tenant) có stack dịch vụ riêng biệt. Trong một số trường hợp thực tế, không phải tất cả tenant đều sẵn sàng nâng cấp lên phiên bản mới cùng lúc (do lý do nghiệp vụ, thời gian triển khai, kiểm thử…).

Do đó, hệ thống cần một chiến lược phiên bản hóa phù hợp để:
- Hỗ trợ đồng thời nhiều phiên bản dịch vụ (per tenant)
- Giảm thiểu rủi ro cập nhật hàng loạt
- Cho phép tenant thử nghiệm trước trên staging

## Quyết định

### 1. Mỗi tenant có thông tin version riêng

- Trong config (hoặc secret) của từng tenant, có trường `stack_version`:

```yaml
tenant_id: "abc"
stack_version: "v1.3.5"
```

* Giá trị này dùng để:

  * Gắn vào log, alert, metric
  * Định danh Docker image được deploy
  * Điều hướng contract test tương ứng

### 2. Pipeline CI/CD hỗ trợ build nhiều version

* Mỗi commit được tag `vX.Y.Z` sẽ:

  * Build Docker image `sub-user:vX.Y.Z`
  * Tạo release notes, changelog tự động
* Pipeline phát hành theo tenant:

  * `dx-deploy --tenant abc --version v1.3.5`

### 3. Tenant staging để kiểm thử trước

* Mỗi tenant có môi trường `staging.tenant-a.dx-vas.vn`
* Tenant có thể thử phiên bản mới tại staging trước khi duyệt lên `prod`

### 4. Superadmin quản lý mapping version

* Superadmin Webapp có bảng mapping:

| Tenant | Version     | Trạng thái |
| ------ | ----------- | ---------- |
| A      | v1.3.5      | Production |
| B      | v1.4.0-beta | Staging    |
| C      | v1.2.9      | Legacy     |

* Cho phép kiểm soát rollback, hotfix riêng cho từng tenant

## Hệ quả

✅ Ưu điểm:

* Cho phép rollout theo cụm hoặc từng tenant
* Giảm rủi ro lỗi diện rộng
* Hỗ trợ A/B testing, backward compatibility dễ dàng
* Tenant chủ động kiểm thử mà không ảnh hưởng sản phẩm chính

⚠️ Lưu ý:

* Quản lý version cần đồng bộ với DB schema → cần kết hợp với `adr-023-schema-migration-strategy.md`
* Tăng độ phức tạp của CI/CD & Monitoring

## Liên kết liên quan

* [`adr-015-deployment-strategy.md`](./adr-015-deployment-strategy.md)
* [`adr-018-release-approval-policy.md`](./adr-018-release-approval-policy.md)
* [`adr-023-schema-migration-strategy.md`](./adr-023-schema-migration-strategy.md)
* [`adr-022-sla-slo-monitoring.md`](./adr-022-sla-slo-monitoring.md)
