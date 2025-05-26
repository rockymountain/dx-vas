# 📘 Interface Contract – User Service (dx-vas)

## 🗭 Mục tiêu

Tài liệu này định nghĩa giao tiếp cho **User Service**, microservice chịu trách nhiệm:

* Quản lý người dùng nội bộ (tài khoản, trạng thái, phân quyền)
* Cung cấp dữ liệu `roles`, `permissions`, `is_active`
* Ghi nhận và cung cấp audit log cho các thao tác quản trị
* Là backend xử lý chính cho các API `/users`, `/roles`, `/permissions`, `/audit/logs` được gọi từ Admin Webapp thông qua Gateway

---

## 📙 Endpoint User Service phơi ra (Provider API)

### 👤 User, Role, Permission APIs

| Method | Endpoint             | Mô tả                         | Input Schema / Query Params          | Output Schema         | Permission Code       |
|--------|----------------------|-------------------------------|--------------------------------------|------------------------|------------------------|
| GET    | `/users`             | Lấy danh sách người dùng      | `page`, `search`, `role?`, `status?` | `List[UserOut]`        | `VIEW_USER_ALL`        |
| POST   | `/users`             | Tạo mới người dùng            | `UserCreate`                         | `UserOut`              | `CREATE_USER`          |
| PATCH  | `/users/{id}`        | Cập nhật thông tin người dùng | `UserUpdate`                         | `UserOut`              | `EDIT_USER`            |
| PATCH  | `/users/{id}/status` | Cập nhật trạng thái hoạt động | `SetActiveRequest`                   | `UserOut`              | `EDIT_USER_STATUS`     |
| GET    | `/roles`             | Danh sách role                | —                                    | `List[RoleOut]`        | `VIEW_ROLE_ALL`        |
| POST   | `/roles`             | Tạo role mới                  | `RoleCreate`                         | `RoleOut`              | `CREATE_ROLE`          |
| GET    | `/permissions`       | Danh sách permission          | —                                    | `List[PermissionOut]`  | `VIEW_PERMISSION_ALL`  |

### 🔗 Role-Permission và User-Role APIs

| Method | Endpoint                                       | Mô tả                        | Input Schema       | Output Schema | Permission Code               |
|--------|------------------------------------------------|------------------------------|--------------------|----------------|-------------------------------|
| POST   | `/roles/{id}/permissions`                      | Gán permission vào role      | `PermissionAssign` | `RoleOut`      | `ASSIGN_PERMISSION_TO_ROLE`   |
| DELETE | `/roles/{role_id}/permissions/{permission_id}` | Thu hồi permission khỏi role | —                  | `RoleOut`      | `REMOVE_PERMISSION_FROM_ROLE` |
| POST   | `/users/{id}/roles`                            | Gán role cho user            | `RoleAssign`       | `UserOut`      | `ASSIGN_ROLE_TO_USER`         |
| DELETE | `/users/{user_id}/roles/{role_id}`             | Thu hồi role khỏi user       | —                  | `UserOut`      | `REMOVE_ROLE_FROM_USER`       |

### 📜 Audit Log API

| Method | Endpoint         | Mô tả                    | Input Schema / Query Params                              | Output Schema       | Permission Code     |
|--------|------------------|--------------------------|-----------------------------------------------------------|----------------------|----------------------|
| GET    | `/audit/logs`    | Lịch sử thao tác hệ thống | `resource: str`, `user_id?: UUID`, `date?: ISODateString` | `List[AuditLogOut]` | `VIEW_AUDIT_LOG`     |

> Endpoint `/audit/logs` dùng để truy vết các hành động quan trọng như tạo/sửa role, permission, user… cho Admin Webapp.

---

## 🔐 Xác thực & RBAC

* Mọi request đi qua API Gateway → JWT bắt buộc
* Gateway forward header:
  * `X-User-ID`, `X-Role`, `X-Permissions`, `Trace-ID`
* Backend chỉ cần kiểm tra permission code có trong `X-Permissions`

---

## 📦 Response structure

Tuân thủ [ADR-012](../ADR/adr-012-response-structure.md):

```json
{
  "data": {...},
  "error": null,
  "meta": {
    "trace_id": "...",
    "timestamp": "...",
    "source": "user_service"
  }
}
```

---

## ⚠️ Lỗi đặc thù

| Code                      | Mô tả                                       |
|---------------------------|---------------------------------------------|
| `USER_NOT_FOUND`          | Không tìm thấy người dùng tương ứng         |
| `ROLE_NOT_FOUND`          | Không tồn tại role                          |
| `PERMISSION_INVALID`      | Permission không hợp lệ hoặc không gắn được |
| `EMAIL_ALREADY_USED`      | Email đã tồn tại trong hệ thống             |
| `INVALID_ROLE_ASSIGNMENT` | Không thể gán role không được phép          |

---

## ✅ Ghi chú

* Permissions được định nghĩa tĩnh theo chuẩn hệ thống (ADR-007), được load vào DB qua migration ban đầu, và **API chỉ cho phép GET /permissions** (không có create/update/delete trực tiếp qua API).
* Các thay đổi gán/thu hồi role-permission, user-role được ghi nhận và audit rõ ràng.
* Khi trạng thái `is_active` thay đổi:
  * Gateway sẽ nhận event từ User Service (qua event bus như Redis Pub/Sub hoặc message queue)
  * Gateway cập nhật cache permission tương ứng hoặc refresh context cho token mới được cấp
  * Trong trường hợp token vẫn còn hiệu lực, backend có thể kiểm tra `X-Permissions` đã được cập nhật đúng nếu cần bảo vệ logic sâu hơn.
* `/audit/logs` phản ánh toàn bộ thao tác có ảnh hưởng tới RBAC, phân quyền, tạo user… từ Admin Webapp
* Người dùng có thể là giáo viên, nhân viên, phụ huynh – được ánh xạ từ SIS/CRM nhưng toàn bộ định danh, trạng thái `is_active` và quyền truy cập (`roles`, `permissions`) được quản lý tập trung tại User Service.
* Mọi thay đổi đều được audit theo [ADR-008]

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)

---

> “User Service là trái tim quản lý định danh – duy trì quyền lực, truy xuất và tính minh bạch cho toàn bộ hệ thống.”
