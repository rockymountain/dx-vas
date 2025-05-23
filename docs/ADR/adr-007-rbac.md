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

## 🧱 Cấu trúc dữ liệu RBAC

### Bảng `roles`

| id | name    |
| -- | ------- |
| 1  | student |
| 2  | teacher |
| 3  | parent  |

### Bảng `permissions`

| id | code                             | description      | resource | action | condition (JSONB)                                                     |
| -- | -------------------------------- | ---------------- | -------- | ------ | --------------------------------------------------------------------- |
| 1  | VIEW\_STUDENT\_SCORE\_OWN\_CHILD | Xem điểm con     | score    | read   | { "accessible\_student\_ids": \["\<student\_id\_of\_their\_child>"] } |
| 2  | CREATE\_COURSE\_GLOBAL           | Tạo mới khoá học | course   | create | null                                                                  |

### Bảng `role_permissions`

| role\_id | permission\_id |
| -------- | -------------- |
| 1        | 1              |
| 2        | 1, 2           |

### Bảng `user_roles`

| user\_id | role\_id |
| -------- | -------- |
| u123     | 3        |

---

## 🔁 Caching & preload

* Redis key: `rbac:user:{user_id}` → danh sách permission object (`code`, `resource`, `action`, `condition`)
* TTL tùy chỉnh (5–15 phút), preload khi login hoặc chạy background task
* Gateway luôn evaluate lại điều kiện theo context → chỉ forward các permission code hợp lệ

---

## 🧩 Tích hợp service khác

* Backend (Notification, CRM Adapter...) sử dụng `X-Permissions`, `X-Role`, `X-User-ID` từ Gateway
* Backend **không cần decode JWT** hoặc re-check permission (trừ khi audit đặc biệt)

---

## ✅ Lợi ích

* Phân quyền động, chính xác đến từng request context
* Cho phép cập nhật permission không cần chỉnh JWT
* Quản lý đơn giản nhờ sử dụng permission code duy nhất, dễ kiểm tra và log

---

## ❌ Rủi ro & Giải pháp

| Rủi ro                                | Giải pháp                                                   |
| ------------------------------------- | ----------------------------------------------------------- |
| Cache Redis không đồng bộ             | TTL ngắn + preload khi login + invalidate khi update        |
| Nhiều role conflict quyền             | Dùng union quyền hoặc định nghĩa rule resolve conflict      |
| Backend thiếu context để check        | Evaluate tại Gateway, hoặc forward context đầy đủ           |
| Condition quá phức tạp, khó kiểm soát | Chuẩn hoá key của `condition`, viết test & tài liệu rõ ràng |

---

## 📎 Tài liệu liên quan

* Auth Strategy: [ADR-006](./adr-006-auth-strategy.md)
* Audit Logging: [ADR-008](./adr-008-audit-logging.md)
* Security Strategy: [ADR-004](./adr-004-security.md)

---

> "RBAC tốt không chỉ kiểm soát quyền – mà còn phản ánh rõ triết lý kiểm soát và tin cậy trong toàn hệ thống."
