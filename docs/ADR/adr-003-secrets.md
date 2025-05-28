---
id: adr-003-secrets
title: ADR-003 - Chiến lược Quản lý và Xoay vòng Secrets cho hệ thống dx-vas
status: accepted
author: DX VAS Security & DevOps Team
date: 2025-05-22
tags: [secrets, security, secret-rotation, dx-vas]
---

# ADR-003: Chính sách quản lý Secrets

## Bối cảnh

Hệ thống dx-vas bao gồm nhiều service (Auth, User, Notification…) và nhiều tenant (trường thành viên), mỗi tenant có thể sử dụng các kênh gửi thông báo riêng biệt như Zalo OA, Gmail API, SMS Provider, v.v.

Tất cả các secret nhạy cảm như token, API key, mật khẩu CSDL... cần được quản lý tập trung, có khả năng rotate, audit truy cập, và phân quyền theo nguyên tắc tối thiểu.

## Quyết định

### 1. Sử dụng Google Cloud Secret Manager

- Tất cả secrets đều được lưu trữ tại Google Cloud Secret Manager (GCSM)
- Mỗi secret có thể có nhiều phiên bản (`versions`)
- Việc truy xuất phải thông qua identity của Cloud Run / Cloud Function hoặc workload identity

### 2. Mỗi tenant có secrets riêng biệt (nếu dùng)

- Tenant có thể cấu hình Zalo OA riêng, Gmail API key riêng hoặc các webhook/token tùy theo nhu cầu
- Những secret này sẽ được lưu theo định danh:

```text
projects/dx-vas-tenant-abc/secrets/zalo-oa-token
projects/dx-vas-tenant-xyz/secrets/gmail-api-key
```

* Sub Notification Service sẽ truy xuất secrets tương ứng theo tenant\_id
* Việc rotate, phân quyền và revoke secret được thực hiện theo từng tenant

### 3. Phân quyền truy cập secrets

* Mỗi service chỉ có quyền truy xuất secrets cần thiết theo nguyên tắc **least privilege**
* Ví dụ:

  * `auth-service-master` có quyền đọc secret `jwt-signing-key`
  * `notification-service-xyz` có quyền đọc `zalo-oa-token` trong project `dx-vas-tenant-xyz`
* Superadmin Webapp có công cụ xem và cập nhật secret cấu hình theo tenant (ghi log audit)

### 4. Rotate và revoke secret

* Mỗi secret nên có TTL định kỳ (VD: 90 ngày)
* Công cụ quản trị có thể trigger rotate → tạo version mới, update config và revoke version cũ
* Notification Service cần hỗ trợ reload token từ Secret Manager mà không cần restart service

## Hệ quả

✅ Ưu điểm:

* Secrets được tách biệt rõ ràng theo môi trường, theo tenant
* Hỗ trợ nhiều cấu hình linh hoạt: mỗi trường có thể dùng kênh gửi riêng
* Có thể rotate mà không gián đoạn dịch vụ
* Dễ audit truy cập theo từng tenant

⚠️ Lưu ý:

* Cần công cụ hoặc script hỗ trợ cập nhật secret cho từng tenant
* Tenant phải chịu trách nhiệm cung cấp và quản lý token riêng nếu không dùng mặc định hệ thống

## Liên kết liên quan

* [`adr-008-audit-logging.md`](./adr-008-audit-logging.md)
* [`adr-015-deployment-strategy.md`](./adr-015-deployment-strategy.md)
* [`README.md#9-quản-lý-bảo-mật--secrets`](../README.md#9-quản-lý-bảo-mật--secrets)

---
> “Không có secret nào nên tồn tại mãi mãi – rotate là cách bạn bảo vệ hệ thống khi bạn quên rằng đã từng để lộ nó.”
