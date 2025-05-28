---
id: adr-004-security
title: ADR-004 - Chiến lược Bảo mật tổng thể cho hệ thống dx-vas
status: accepted
author: DX VAS Security Team
date: 2025-05-22
tags: [security, hardening, dx-vas, cloud-run, jwt, rate-limiting]
---

# ADR-004: Chính sách Bảo mật Hệ thống (Security Policy)

## Bối cảnh

Hệ thống dx-vas cung cấp dịch vụ cho nhiều trường thành viên khác nhau (multi-tenant), bao gồm nhiều loại người dùng: học sinh, giáo viên, nhân viên, phụ huynh, quản trị viên...

Bảo mật hệ thống không chỉ bao gồm xác thực và phân quyền, mà còn đảm bảo:

- Mỗi tenant được cách ly độc lập
- Không có rò rỉ dữ liệu giữa các tenant
- Dữ liệu nhạy cảm được mã hóa và giám sát truy cập
- JWT và Redis cache được sử dụng an toàn

## Quyết định

### 1. Xác thực

- Sử dụng Google OAuth2 cho nhân viên & giáo viên
- Sử dụng OTP login (SMS/email) cho phụ huynh và học sinh
- Tất cả xác thực đều đi qua Auth Master hoặc Sub Auth Service (theo từng tenant)

### 2. Phân quyền động (RBAC)

- JWT chứa `user_id`, `tenant_id`, `roles`
- API Gateway sẽ lấy `permissions` từ Redis cache (`rbac:{user_id}:{tenant_id}`)
- Hệ thống đánh giá `condition` động dựa trên context của user/request

---

### 3. Cách ly Tenant (Tenant Isolation)

Hệ thống đảm bảo cách ly tenant ở nhiều cấp độ:

| Lớp | Chi tiết |
|-----|----------|
| **Hạ tầng (GCP)** | Mỗi tenant có GCP project riêng (`dx-vas-tenant-*`), không chia sẻ Cloud Run, Cloud SQL, Secret |
| **Mạng** | Các service tenant chỉ nhận request từ Gateway hoặc IP danh sách cho phép |
| **Dữ liệu** | Mỗi tenant có thể có **Cloud SQL instance riêng** (trong project `dx-vas-tenant-*` hoặc `dx-vas-data`). Trường hợp dùng chung instance → phải đảm bảo cách ly bằng schema riêng, hoặc ít nhất các bảng có `tenant_id` rõ ràng. |
| **JWT** | Mỗi tenant sử dụng JWT chứa `tenant_id`, giúp Gateway xác định context chính xác |
| **CI/CD** | Pipeline tách biệt, không có khả năng cross-deploy giữa tenants |

---

### 4. Ranh giới Tin cậy JWT (JWT Trust Boundary)

Để đảm bảo an toàn trong xác thực và phân quyền:

- Tất cả JWT đều phải:
  - Có chữ ký với secret/key được quản lý bởi Gateway
  - Có `tenant_id` để xác định context phân quyền
- **Gateway là nơi duy nhất tin cậy để đọc & phân tích JWT**
- Các service backend (Sub Service, Core Service) không được xử lý trực tiếp RBAC nếu không xác minh JWT qua Gateway
- Tất cả JWT đều có TTL ngắn (dưới 1 giờ), và có thể revoke nếu cần

> Nếu cần xác thực giữa các service, phải sử dụng `X-Internal-Call` hoặc service token nội bộ có kiểm soát

---

## Hệ quả

✅ Ưu điểm:

- Đảm bảo mỗi tenant được cách ly tuyệt đối về dữ liệu, mạng, quyền
- Ngăn chặn tấn công ngang giữa tenant
- Giảm thiểu rủi ro liên quan đến token giả, cache sai context
- Dễ mở rộng mô hình bảo mật theo từng lớp (network, ứng dụng, logic)

⚠️ Lưu ý:

- Việc revoke JWT tức thời cần triển khai thêm token blacklist hoặc short TTL
- Redis cache cần có namespace rõ ràng theo `tenant_id`, ví dụ: `rbac:{user_id}:{tenant_id}` (xem chi tiết trong [`adr-007-rbac.md`](./adr-007-rbac.md))

## Liên kết liên quan

- [`adr-006-auth-strategy.md`](./adr-006-auth-strategy.md)
- [`adr-007-rbac.md`](./adr-007-rbac.md)
- [`adr-008-audit-logging.md`](./adr-008-audit-logging.md)
- [`rbac-deep-dive.md`](../architecture/rbac-deep-dive.md)

---
> “Security không phải là một module – mà là mindset toàn hệ thống.”
