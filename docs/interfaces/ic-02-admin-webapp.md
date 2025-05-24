# 📘 Interface Contract – Admin Webapp (dx\_vas)

## 🧭 Mục tiêu

Tài liệu này mô tả các API mà **Admin Webapp** sử dụng thông qua API Gateway, nhằm:

* Định nghĩa rõ từng endpoint, schema input/output
* Xác định permission code yêu cầu (theo ADR-007)
* Đồng bộ hóa với các nguyên tắc từ API Gateway và RBAC hệ thống

---

## 🧩 Danh sách endpoint tiêu biểu

| Method | Endpoint              | Mô tả                         | Input Schema / Query Params                             | Output Schema         | Permission Code         |
| ------ | --------------------- | ----------------------------- | ------------------------------------------------------- | --------------------- | ----------------------- |
| GET    | `/users`              | Lấy danh sách người dùng      | `page: int`, `search: str`                              | `List[UserOut]`       | `VIEW_USER_ALL`         |
| POST   | `/users`              | Tạo người dùng mới            | `UserCreate`                                            | `UserOut`             | `CREATE_USER`           |
| PATCH  | `/users/{id}`         | Cập nhật thông tin người dùng | `UserUpdate`                                            | `UserOut`             | `EDIT_USER`             |
| GET    | `/roles`              | Danh sách role                | —                                                       | `List[RoleOut]`       | `VIEW_ROLE_ALL`         |
| POST   | `/roles`              | Tạo role                      | `RoleCreate`                                            | `RoleOut`             | `CREATE_ROLE`           |
| GET    | `/permissions`        | Danh sách permission          | —                                                       | `List[PermissionOut]` | `VIEW_PERMISSION_ALL`   |
| POST   | `/classes`            | Tạo lớp học                   | `ClassCreate`                                           | `ClassOut`            | `CREATE_CLASS`          |
| GET    | `/audit/logs`         | Lịch sử thao tác              | `resource: str`, `user_id: UUID`, `date: ISODateString` | `List[AuditLogOut]`   | `VIEW_AUDIT_LOG`        |
| POST   | `/notifications/send` | Gửi thông báo                 | `NotificationRequest`                                   | `NotificationResult`  | `SEND_NOTIFICATION_ALL` |

> Các schema (UserCreate, RoleOut, ...) được định nghĩa thống nhất bằng Pydantic hoặc JSON Schema và phải đồng bộ với OpenAPI.

---

## 🔐 Cơ chế xác thực & phân quyền

* Tất cả request từ Admin Webapp gọi qua API Gateway → đều phải gửi `Authorization: Bearer <token>`.
* API Gateway sẽ forward các header:

  * `X-User-ID`, `X-Role`, `X-Auth-Method`, `X-Permissions`, `Trace-ID`
* Backend kiểm tra sự tồn tại của permission code trong `X-Permissions`, không cần re-evaluate condition (theo [ADR-007](../ADR/adr-007-rbac.md))

---

## 📦 Response structure

Tuân thủ toàn bộ theo [ADR-012](../ADR/adr-012-response-structure.md):

```json
{
  "data": {...},
  "error": null,
  "meta": {
    "timestamp": "...",
    "trace_id": "...",
    "source": "api_gateway"
  }
}
```

Khi lỗi:

```json
{
  "data": null,
  "error": {
    "code": "FORBIDDEN",
    "message": "Không đủ quyền truy cập"
  },
  "meta": {
    "timestamp": "...",
    "trace_id": "..."
  }
}
```

---

## ✅ Ghi chú

* UI của Admin Webapp phải hiện thông báo phù hợp dựa vào `error.code`, `error.message` (theo [ADR-011](../ADR/adr-011-api-error-format.md))
* Khi thao tác thay đổi dữ liệu (user, class, role, permission...), backend phải tạo audit log (theo [ADR-008](../ADR/adr-008-audit-logging.md))
* Admin Webapp **không tạo trace\_id** – mà log và truyền lại trace\_id đã nhận trong response để quan sát xuyên suốt chuỗi call (hoặc để debug frontend/backend tương ứng)

---

## 📎 Tài liệu liên quan

* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [ADR-011: API Error Format](../ADR/adr-011-api-error-format.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)
* [README.md – API Gateway & RBAC](../README.md)

---

> “Admin Webapp không chỉ là nơi thao tác – mà là nơi thể hiện rõ luồng quyền lực và trách nhiệm trong hệ thống.”
