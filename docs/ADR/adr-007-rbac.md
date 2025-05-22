---
id: adr-007-rbac
title: ADR-007: Chiến lược Phân quyền dựa trên Vai trò (RBAC Strategy) cho hệ thống dx_vas
status: accepted
author: DX VAS Security Team
date: 2025-06-22
tags: [rbac, access-control, roles, permissions, dx_vas]
---

## 📌 Bối cảnh

Hệ thống **dx_vas** phục vụ nhiều loại người dùng và dịch vụ:
- Học sinh, giáo viên, phụ huynh (qua Portal/LMS)
- Quản trị viên, nhân viên trường (qua CRM, SIS)
- Dịch vụ nội bộ gọi API qua Gateway (LMS Adapter, CRM Proxy...)

Với quy mô nhiều dịch vụ và người dùng như vậy, cần một chiến lược phân quyền linh hoạt, thống nhất và dễ mở rộng, giúp:
- Giới hạn truy cập tài nguyên theo vai trò và quyền
- Kiểm soát hành vi người dùng và dịch vụ rõ ràng
- Đồng bộ cơ chế kiểm tra quyền ở nhiều thành phần hệ thống

## 🧠 Quyết định

**Áp dụng chiến lược RBAC động, lưu trữ trung tâm và cache hiệu quả, với Role/Permission rõ ràng, hỗ trợ kiểm tra tại Gateway và/hoặc các service backend.**

## 🧩 Mô hình dữ liệu RBAC

### 1. Các thực thể chính
- `User`: thực thể đăng nhập (người dùng hoặc dịch vụ)
- `Role`: định danh vai trò (student, teacher, admin, service)
- `Permission`: quyền cụ thể (ví dụ: `student_info:read`, `grades:update`)

### 2. Quan hệ
- Một `User` có thể có **nhiều `Role`**
- Một `Role` có thể có **nhiều `Permission`**
- `Permissions` nên được định danh theo định dạng: `resource:action`

```text
User → Role → Permissions
        ↘         ↘
      (student)   (student_info:read, attendance:view)
```

### 3. Lưu trữ
- Roles, Permissions và Mapping lưu trong DB (PostgreSQL, qua API Gateway)
- Dữ liệu cache tại Redis: `user_id` → `permissions[]`
- Hỗ trợ preload: pattern route (e.g., `/students/:id:GET`) → permission

## 🔐 Quy trình kiểm tra quyền

### ✅ Tại API Gateway
- JWT xác thực → chứa `sub`, `role`
- Gateway dùng `path + method` → map sang `required_permission`
- Lấy quyền user từ Redis (cache theo `user_id`)
- Nếu `required_permission ∈ user_permissions` → cho phép
- Nếu không → trả 403

### ✅ Tại Backend Services
Có 3 mô hình triển khai:

#### **Option 1: Tin tưởng Gateway (Header-based)**
- Gateway forward các header:
  - `X-User-Id`, `X-Role`, `X-Permissions`
- Backend chỉ cần check logic nghiệp vụ → không cần re-verify JWT

#### **Option 2: Tự kiểm tra (Verify JWT + Redis)**
- Backend service tự verify JWT (nếu dùng public key)
- Truy vấn Redis/DB để lấy `permissions`
- Tự kiểm tra hành động có được phép không

#### **Option 3: Hybrid**
- Tin `X-Permissions`, nhưng fallback Redis nếu thiếu
- Xác thực kỹ hơn với endpoint quan trọng (e.g., `PUT /grades`)

### Gợi ý triển khai theo trust level:
- CRM Adapter (nội bộ) → Option 1
- LMS Proxy → Option 2 hoặc 3
- Notification Service → Option 1

## ⚙️ Cache & Hiệu năng
- Redis key: `rbac:user:{user_id}` → list permissions (TTL 5–15 phút)
- Redis key: `rbac:pattern:{path}:{method}` → permission cần thiết
- Invalidate cache khi thay đổi role/permission/user (bằng pub/sub hoặc tag key)

## 🛠 Quản lý Roles/Permissions
- Giao diện quản trị phân quyền nằm trong Admin Webapp
- API Gateway expose endpoint quản lý RBAC:
  - `GET /roles`, `GET /permissions`, `PUT /user-role`, v.v.
- Log lại mọi hành động RBAC (xem ADR Audit Logging)

## ✅ Lợi ích

- Phân quyền linh hoạt, dễ mở rộng, không hardcode
- Kiểm soát rõ ràng hành vi truy cập từng user/service
- Cho phép backend lựa chọn mức tin tưởng phù hợp
- Tăng khả năng kiểm toán và trace lỗi

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Cache lỗi → quyền sai | TTL ngắn + invalidate bằng pub/sub |
| Backend tin tưởng sai header | Áp dụng theo trust boundary rõ ràng + JWT fallback |
| Người dùng thay đổi role nhưng chưa được update | Cache bust ngay khi cập nhật role/permission |

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Hardcode quyền trong mã nguồn | Khó bảo trì, không thay đổi runtime |
| ACL riêng từng service | Mất đồng bộ, phức tạp khi mở rộng |
| Kiểm tra quyền chỉ dựa vào role | Không đủ chi tiết cho hành vi truy cập cụ thể |

## 📎 Tài liệu liên quan

- Auth Strategy: [ADR-006](./adr-006-auth-strategy.md)
- Security Hardening: [ADR-004](./adr-004-security.md)
- Audit Logging: [ADR-008](./adr-008-audit-logging.md)

---
> “RBAC tốt không chỉ kiểm soát quyền – mà còn phản ánh rõ triết lý kiểm soát và tin cậy trong toàn hệ thống.”
