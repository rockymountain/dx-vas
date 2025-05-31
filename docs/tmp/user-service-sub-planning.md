# 📘 Kế hoạch thiết kế `user-service/sub/`

## 🧩 1. `design.md`: Mô tả thiết kế tổng thể

### 🎯 Mục tiêu
- Quản lý và cung cấp dữ liệu người dùng cục bộ cho một tenant
- Không tạo/sửa/xoá user – chỉ "consume" dữ liệu từ `user-service/master`
- Đảm bảo đọc-only, hỗ trợ kiểm tra quyền của user trong từng tenant

### 🔧 Thành phần chính

| Thành phần        | Vai trò                                                                 |
|-------------------|-------------------------------------------------------------------------|
| `UserLocal`       | Bản sao của `UserGlobal`, dùng để hiển thị danh sách hoặc profile       |
| `RBACResolver`    | Tính toán quyền hiện tại của user trong tenant từ role template         |
| `RoleTemplateLite`| Bảng cache role template từ Master                                      |
| `PermissionTemplateLite` | Bảng cache permission template từ Master                        |

### 🔄 Luồng dữ liệu

- Sub service **consume các sự kiện** từ master:
  - `user_global_created`, `user_updated`
  - `user_assigned_to_tenant`, `user_removed_from_tenant`
  - `rbac_template_updated`
- Tự động đồng bộ vào local DB
- API chỉ expose các luồng READ

---

## 🧩 2. `data-model.md`: Mô hình dữ liệu nội bộ

### 🔹 `UserLocal`

| Trường               | Kiểu     | Ghi chú                                                           |
|----------------------|----------|-------------------------------------------------------------------|
| `user_id`            | string   | ID đồng bộ từ UserGlobal                                          |
| `email`              | string   |                                                                   |
| `full_name`          | string   |                                                                   |
| `auth_provider`      | enum     | [local, google]                                                   |
| `status`             | enum     | Bản sao từ UserGlobal.status (`active`, `suspended`, ...)         |
| `created_at`         | datetime |                                                                   |
| `updated_at`         | datetime |                                                                   |
| `is_active_in_tenant`| boolean  | Tổng hợp từ `status` và `assignment_status`                      |

> 🧠 `is_active_in_tenant` là field **tính toán hoặc cache**, không đồng bộ trực tiếp từ master.

---

### 🔹 `UserTenantRole`

| Trường        | Kiểu     | Ghi chú                         |
|---------------|----------|---------------------------------|
| `user_id`     | string   |                                 |
| `role_code`   | string   |                                 |
| `permissions` | string[] | Expand từ role_code             |

> ❌ `tenant_id` bị loại bỏ vì mỗi Sub Service chỉ phục vụ 1 tenant duy nhất.

---

### 🔹 `RoleTemplateLite`

| Trường        | Kiểu     | Ghi chú                         |
|---------------|----------|---------------------------------|
| `role_code`   | string   |                                 |
| `name`        | string   |                                 |
| `description` | string   |                                 |

---

### 🔹 `PermissionTemplateLite`

| Trường        | Kiểu     | Ghi chú                         |
|---------------|----------|---------------------------------|
| `code`        | string   |                                 |
| `resource`    | string   |                                 |
| `action`      | string   |                                 |
| `description` | string   |                                 |

---

## 🧩 3. `interface-contract.md`: Đặc tả API & metadata

### 📚 API chính

| Method | Path                        | Tác dụng                            |
|--------|-----------------------------|-------------------------------------|
| GET    | `/users`                    | Danh sách user trong tenant         |
| GET    | `/users/me`                | Thông tin user hiện tại (token)     |
| GET    | `/users/me/permissions`    | Các quyền của user hiện tại         |
| GET    | `/roles`                   | Danh sách role từ template (local)  |
| GET    | `/permissions`             | Danh sách permission từ template    |

### 📌 Metadata & behavior

- `x-required-permission`: chỉ áp dụng nếu cần bảo vệ endpoint `GET /users`
- `x-audit-action`: gợi ý thêm vào `/me` hoặc `/permissions`
- `x-emits-event`: **không dùng** tại sub (chỉ consume event)

---

## 🧩 4. `openapi.yaml`: Mô tả chi tiết API (chuẩn 5★)

### 🧱 Schema chính

| Tên Schema              | Mô tả                                |
|-------------------------|---------------------------------------|
| `UserLocal`             | Thông tin user local trong tenant     |
| `UserMeResponse`        | Thông tin người dùng hiện tại         |
| `PermissionListResponse`| Danh sách quyền                      |
| `RoleTemplateLite`      | Role đồng bộ từ master                |

### 📐 Quy ước

- Tất cả response dùng: `meta`, `data`, `error` (ADR-012)
- Tất cả schema có: `description`, `example`, `readOnly/writeOnly` đúng chuẩn
- Không có method `POST`, `PATCH`, `DELETE`
- Enum `status`, `auth_provider` dùng `$ref` từ `components/schemas`
- Tags: `users`, `me`, `roles`, `permissions`

```yaml
tags:
  - name: users
    description: Tra cứu danh sách user trong tenant
  - name: me
    description: Dữ liệu người dùng hiện tại
  - name: roles
    description: Vai trò (RBAC template local)
  - name: permissions
    description: Quyền (RBAC template local)
```

---

## ✅ Tổng kết

* Đã đồng bộ hoàn toàn với kiến trúc phân tầng Master/Sub
* Tuân thủ các ADR chính: 012, 017, 025, 027
* Tách rõ vùng dữ liệu, không bị lặp chức năng với `user-service/master`
* Đảm bảo performance và độc lập thông qua event-consume + local cache
