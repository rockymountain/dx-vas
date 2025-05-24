# 📘 Interface Contract – User Service (dx\_vas)

## 🧭 Mục tiêu

Tài liệu này định nghĩa giao tiếp cho **User Service**, microservice chịu trách nhiệm:

* Quản lý người dùng nội bộ (tài khoản, trạng thái, phân quyền)
* Cung cấp dữ liệu `roles`, `permissions`, `is_active`
* Là backend xử lý chính cho các API `/users`, `/roles`, `/permissions` được gọi từ Admin Webapp thông qua Gateway

---

## 🧩 Endpoint User Service phơi ra (Provider API)

### 📁 User, Role, Permission APIs

| Method | Endpoint             | Mô tả                         | Input Schema / Query Params          | Output Schema         | Permission Code       |
| ------ | -------------------- | ----------------------------- | ------------------------------------ | --------------------- | --------------------- |
| GET    | `/users`             | Lấy danh sách người dùng      | `page`, `search`, `role?`, `status?` | `List[UserOut]`       | `VIEW_USER_ALL`       |
| POST   | `/users`             | Tạo mới người dùng            | `UserCreate`                         | `UserOut`             | `CREATE_USER`         |
| PATCH  | `/users/{id}`        | Cập nhật thông tin người dùng | `UserUpdate`                         | `UserOut`             | `EDIT_USER`           |
| PATCH  | `/users/{id}/status` | Cập nhật trạng thái hoạt động | `SetActiveRequest`                   | `UserOut`             | `EDIT_USER_STATUS`    |
| GET    | `/roles`             | Danh sách role                | —                                    | `List[RoleOut]`       | `VIEW_ROLE_ALL`       |
| POST   | `/roles`             | Tạo role mới                  | `RoleCreate`                         | `RoleOut`             | `CREATE_ROLE`         |
| GET    | `/permissions`       | Danh sách permission          | —                                    | `List[PermissionOut]` | `VIEW_PERMISSION_ALL` |

---

### 🔗 Role-Permission và User-Role APIs

| Method | Endpoint                                       | Mô tả                        | Input Schema       | Output Schema | Permission Code               |
| ------ | ---------------------------------------------- | ---------------------------- | ------------------ | ------------- | ----------------------------- |
| POST   | `/roles/{id}/permissions`                      | Gán permission vào role      | `PermissionAssign` | `RoleOut`     | `ASSIGN_PERMISSION_TO_ROLE`   |
| DELETE | `/roles/{role_id}/permissions/{permission_id}` | Thu hồi permission khỏi role | —                  | `RoleOut`     | `REMOVE_PERMISSION_FROM_ROLE` |
| POST   | `/users/{id}/roles`                            | Gán role cho user            | `RoleAssign`       | `UserOut`     | `ASSIGN_ROLE_TO_USER`         |
| DELETE | `/users/{user_id}/roles/{role_id}`             | Thu hồi role khỏi user       | —                  | `UserOut`     | `REMOVE_ROLE_FROM_USER`       |

> Permissions trong hệ thống có thể được định nghĩa sẵn hoặc mở rộng động tuỳ theo chiến lược RBAC. Hiện tại, hệ thống **cho phép gán và thu hồi permission động**, nhưng việc tạo permission mới chỉ được thực hiện nội bộ qua migration (không expose API tạo permission động).

* Mọi request đi qua API Gateway → JWT bắt buộc
* Gateway forward header:

  * `X-User-ID`, `X-Role`, `X-Permissions`, `Trace-ID`
* User Service **không decode JWT**, chỉ sử dụng header để kiểm tra quyền truy cập

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
| ------------------------- | ------------------------------------------- |
| `USER_NOT_FOUND`          | Không tìm thấy người dùng tương ứng         |
| `ROLE_NOT_FOUND`          | Không tồn tại role                          |
| `PERMISSION_INVALID`      | Permission không hợp lệ hoặc không gắn được |
| `EMAIL_ALREADY_USED`      | Email đã tồn tại trong hệ thống             |
| `INVALID_ROLE_ASSIGNMENT` | Không thể gán role không được phép          |

---

## ✅ Ghi chú

* Permissions được định nghĩa tĩnh theo chuẩn hệ thống (ADR-007) và được load vào DB qua migration ban đầu. API chỉ cho phép `GET /permissions` (không có create/update/delete trực tiếp qua API).

* Các thay đổi gán/thu hồi role-permission, user-role được ghi nhận và audit rõ ràng.

* Khi trạng thái `is_active` thay đổi:

  * Gateway sẽ nhận event từ User Service (qua event bus như Redis Pub/Sub hoặc message queue).
  * Gateway cập nhật cache permission tương ứng hoặc refresh context cho token mới được cấp.
  * Trong trường hợp token vẫn còn hiệu lực, backend có thể kiểm tra `X-Permissions` đã được cập nhật đúng nếu cần bảo vệ logic sâu hơn.

* Người dùng có thể là giáo viên, nhân viên, phụ huynh (được ánh xạ từ SIS/CRM nhưng tài khoản nằm ở đây)

* Mọi thay đổi trạng thái `is_active` được propagate tới Gateway cache và ảnh hưởng phân quyền tức thời

* Mọi thay đổi tạo/sửa/xoá role & permission đều được audit (theo ADR-008)

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)

---

> “User Service là trái tim quản lý định danh – duy trì quyền lực, truy xuất và tính minh bạch cho toàn bộ hệ thống.”
