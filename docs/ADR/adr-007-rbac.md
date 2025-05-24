---
id: adr-007-rbac
status: accepted
title: ADR-007: Chiến lược phân quyền RBAC động cho toàn hệ thống dx_vas
author: DX VAS Platform Team
date: 2025-06-22
tags: [rbac, security, auth, dx_vas]
---

## 📌 Bối cảnh

Hệ thống `dx_vas` phục vụ nhiều loại người dùng (học sinh, giáo viên, phụ huynh, quản trị viên), mỗi nhóm có quyền khác nhau trên các tài nguyên. Các dịch vụ API hoạt động theo mô hình microservice nên cần cơ chế phân quyền đồng bộ và linh hoạt giữa các service.

---

## 🧠 Quyết định

Áp dụng mô hình **RBAC động**:

* Mỗi user có thể được gán một hoặc nhiều **role**
* Mỗi **role** ánh xạ tới một danh sách **permission** (theo định dạng `resource:action` + `condition`)
* Permission được lưu trong DB và cache tại Redis
* Quyền truy cập được kiểm tra dựa trên cặp `(user_id, path:method)` tại API Gateway hoặc Backend

> JWT **KHÔNG** chứa danh sách `permissions`. Gateway sẽ lấy `role` từ JWT, tra `permissions` từ Redis hoặc DB, evaluate điều kiện dựa trên context, và chỉ forward **các permission hợp lệ (đã evaluate)** qua header `X-Permissions` dưới dạng danh sách các **permission code**.

---

## 🔐 Cách hoạt động

1. User đăng nhập → nhận JWT chứa `sub`, `role`, `auth_method`
2. Khi request đến API Gateway:

   * JWT được xác thực (ADR-006)
   * `role` và `user_id` được extract từ JWT
   * Gateway tra `permissions` từ Redis hoặc DB (gồm cả `condition`)
   * Gateway kiểm tra `resource`, `action` và evaluate `condition` theo context request hiện tại
   * Nếu pass, Gateway forward header:

```http
X-User-ID: u123
X-Role: parent
X-Auth-Method: otp
X-Permissions: VIEW_STUDENT_SCORE_OWN_CHILD, CREATE_COURSE_GLOBAL
```

> `X-Permissions` là danh sách **permission code** đã được Gateway evaluate và xác định là hợp lệ cho request hiện tại. Đây là định danh duy nhất, ngắn gọn và dễ xử lý tại backend.


---

## 🧩 Mô hình dữ liệu RBAC

* `users`: đại diện cho tài khoản hệ thống, được liên kết với actor (giáo viên, phụ huynh...)
* `roles`: nhóm quyền, có thể được gán cho user
* `permissions`: quyền cụ thể, định danh duy nhất bởi `code`, có thể kèm điều kiện JSONB
* `user_roles`: ánh xạ giữa user và role
* `role_permissions`: ánh xạ giữa role và permission

### 🧱 Cấu trúc permission

```json
{
  "code": "VIEW_STUDENT_SCORE_OWN_CHILD",
  "resource": "student",
  "action": "view",
  "condition": {
    "accessible_student_ids": ["student_id_con"]
  },
  "description": "Cho phép xem điểm số của con mình"
}
```

---

## 🧩 Gateway Evaluate Flow

1. Xác thực JWT (hoặc OTP) → lấy `user_id`
2. Kiểm tra `is_active` từ bảng `users` → nếu `false` → từ chối truy cập (`IS_INACTIVE_USER`)
3. Lấy role, gán permission từ Redis cache (`RBAC:{user_id}`) → nếu miss cache thì fallback DB
4. Evaluate condition trong permission (nếu có)
5. Forward header:

   * `X-User-ID`
   * `X-Role`
   * `X-Permissions`: danh sách các permission `code` đã evaluate pass
   * `Trace-ID`

> Gateway không cần biết actor là ai – chỉ dùng `user_id`, và permission code đã được xử lý từ User Service.

---

## 🔧 Cơ chế propagate RBAC

* Các sự kiện sau sẽ kích hoạt propagate:

  * User bị khoá (`is_active = false`)
  * Gán/thu hồi role
  * Gán/thu hồi permission
* Gateway sử dụng Redis Pub/Sub hoặc message queue để nhận event cập nhật cache RBAC
* TTL của Redis cache: 5–15 phút hoặc làm mới theo push event
* Trong trường hợp đặc biệt, backend có thể force-refresh RBAC bằng header debug

---

## 🧩 Quản trị RBAC thông qua User Service

* Tất cả thực thể `users`, `roles`, `permissions` được quản lý bởi **User Service**
* Các API quản trị bao gồm:

  * `/users`, `/users/{id}/roles`, `/users/{id}/status`
  * `/roles`, `/roles/{id}/permissions`
  * `/permissions` (read-only)
* Permissions **được định nghĩa tĩnh**, load vào DB qua migration – không cho phép tạo/sửa/xoá permission qua API (chỉ `GET /permissions` được expose)

---

## 🧭 Service-to-Service RBAC

* Các service như CRM Adapter có thể dùng JWT riêng hoặc credential đặc biệt
* Được gán `X-Role: system` và permission như user thường
* Gateway evaluate như user bình thường nhưng với role là `system`
* Permission cần được cấp cụ thể cho service đó trong bảng RBAC (user\_id đại diện cho service identity)

---

## ⚠️ Rủi ro & Giải pháp

| Rủi ro                                                | Giải pháp                                                                          |
| ----------------------------------------------------- | ---------------------------------------------------------------------------------- |
| Dữ liệu RBAC thay đổi nhưng Gateway vẫn dùng cache cũ | Sử dụng TTL + push event qua Redis Pub/Sub                                         |
| Permission evaluate sai do condition phức tạp         | Chuẩn hoá schema điều kiện và test bằng unit test + integration test               |
| Người dùng bị khoá nhưng token vẫn hợp lệ             | Check `is_active` tại bước evaluate, invalidate RBAC nếu cần                       |
| Service-to-service không được kiểm soát phân quyền    | Phân biệt rõ role system, và permission của service cũng cần được cấu hình rõ ràng |

---

## 📎 Tài liệu liên quan

* [User Service – Interface Contract](../interfaces/user-service-interface-contract.md)
* Auth Strategy: [ADR-006](./adr-006-auth-strategy.md)
* Audit Logging: [ADR-008](./adr-008-audit-logging.md)
* Security Strategy: [ADR-004](./adr-004-security.md)

---

> “RBAC động không chỉ là quyền – mà là bức tranh toàn cảnh về vai trò, điều kiện, và hành vi hệ thống.”
