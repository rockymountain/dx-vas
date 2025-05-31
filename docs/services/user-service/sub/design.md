# 📘 Thiết kế chi tiết `user-service/sub/`

---

## 1. 🧭 Phạm vi và Trách nhiệm (Scope & Responsibilities)

### 🎯 Mục tiêu
- Cung cấp dữ liệu người dùng nội bộ cho frontend trong từng tenant.
- Không thực hiện bất kỳ hành động ghi nào vào dữ liệu người dùng.
- Toàn bộ dữ liệu được đồng bộ từ `user-service/master` thông qua cơ chế event-driven.

### 📦 Các thực thể dữ liệu quản lý
| Thực thể              | Mô tả                                                                 |
|-----------------------|----------------------------------------------------------------------|
| `UserLocal`           | Bản sao của `UserGlobal`, lưu trạng thái người dùng trong tenant     |
| `UserTenantRole`      | Vai trò mà người dùng được gán trong tenant                         |
| `RoleTemplateLite`    | Danh sách role được đồng bộ từ master                                |
| `PermissionTemplateLite` | Danh sách quyền được đồng bộ từ master                           |

---

## 2. 🌐 Thiết kế API chi tiết (Interface Contract)

| Method | Path                     | Tác vụ                        | Yêu cầu permission             |
|--------|--------------------------|-------------------------------|-------------------------------|
| GET    | `/users`                 | Danh sách user trong tenant   | ✅ `tenant.read_users`         |
| GET    | `/users/me`              | Thông tin người dùng hiện tại | ❌                             |
| GET    | `/users/me/permissions`  | Danh sách quyền của user      | ❌                             |
| GET    | `/roles`                 | Role template hiện có         | ✅ `tenant.view_rbac_config`   |
| GET    | `/permissions`           | Permission template hiện có   | ✅ `tenant.view_rbac_config`   |

> 🔧 API dùng chuẩn OpenAPI, tuân thủ cấu trúc response ADR-012, định nghĩa schema riêng cho tất cả response và error.

---

## 3. 🗃️ Mô hình dữ liệu chi tiết (Data Model)

### Bảng: `UserLocal`
| Cột                 | Kiểu     | Ghi chú                                |
|---------------------|----------|----------------------------------------|
| `user_id`           | UUID     | Primary key                            |
| `email`             | string   |                                        |
| `full_name`         | string   |                                        |
| `auth_provider`     | enum     | [local, google]                        |
| `status`            | enum     | Mirror từ UserGlobal                   |
| `is_active_in_tenant` | boolean | Tính từ `assignment_status` & `status`|
| `created_at`        | datetime |                                        |
| `updated_at`        | datetime |                                        |

### Bảng: `UserTenantRole`
| Cột           | Kiểu     | Ghi chú                         |
|---------------|----------|---------------------------------|
| `user_id`     | UUID     |                                 |
| `role_code`   | string   |                                 |
| `permissions` | string[] | Expand từ `role_code`           |

### Bảng: `RoleTemplateLite`
| `role_code`   | string | Primary key |
| `name`        | string |             |
| `description` | string |             |

### Bảng: `PermissionTemplateLite`
| `code`        | string | Primary key |
| `resource`    | string |             |
| `action`      | string |             |
| `description` | string |             |

---

## 4. 🔄 Luồng xử lý nghiệp vụ chính (Business Logic Flows)

### `GET /users/me/permissions`
```mermaid
sequenceDiagram
  participant FE
  participant API
  participant RBACResolver
  participant DB

  FE->>API: GET /users/me/permissions
  API->>RBACResolver: getPermissions(user_id)
  RBACResolver->>DB: Fetch role_code(s) for user
  RBACResolver->>DB: Expand role_code → permission list
  RBACResolver-->>API: permission[]
  API-->>FE: 200 OK + permissions
```

---

## 5. 📣 Các sự kiện Pub/Sub

| Sự kiện nhận                        | Hành động tại Sub Service                                             |
| ----------------------------------- | --------------------------------------------------------------------- |
| `user_global_created`               | Insert `UserLocal`                                                    |
| `user_updated`                      | Update `UserLocal`                                                    |
| `user_assigned_to_tenant`           | Insert/Update `UserTenantRole`, đánh dấu `is_active_in_tenant = true` |
| `user_removed_from_tenant`          | Cập nhật `is_active_in_tenant = false`                                |
| `purge_user_from_tenant` (tuỳ chọn) | Xoá vật lý `UserLocal` nếu chính sách cho phép                        |
| `rbac_template_updated`             | Cập nhật bảng `RoleTemplateLite`, `PermissionTemplateLite`            |

---

## 6. 🔐 Bảo mật & Phân quyền

* Auth: sử dụng JWT token cấp tenant.
* Các API `/users`, `/roles`, `/permissions` **khai báo `x-required-permission` rõ ràng**, nhưng không tự kiểm tra.
* Việc thực thi (`enforce`) quyền sẽ do **API Gateway đảm nhiệm** dựa trên JWT và RBAC cache.
* Các API `/me`, `/me/permissions` chỉ cần token hợp lệ, không cần thêm permission.

---

## 7. ⚙️ Cấu hình & Phụ thuộc (Dependencies)

| Thành phần         | Mục đích                            |
| ------------------ | ----------------------------------- |
| `PG_HOST`, `PG_DB` | Kết nối database                    |
| `KAFKA_BROKER`     | Lắng nghe sự kiện từ Master         |
| `TENANT_ID`        | Gán cứng trong mỗi instance của Sub |
| `JWT_SECRET`       | Xác thực token                      |

> Sub-service này **không gọi trực tiếp service khác**, chỉ consume event.

---

## 8. 🧪 Testing

### 🔹 Unit Test

* RBACResolver.expand()
* Mappers & converters DB → API response
* Schema validator cho OpenAPI

### 🔹 Integration Test

* Simulate event từ Master → assert DB update
* Test endpoint `/me/permissions` trả về chính xác với nhiều role

> 🧪 Có thể sử dụng tools như `pytest`, `testcontainers`, hoặc `async-kafka` mock để kiểm tra event flow

---
