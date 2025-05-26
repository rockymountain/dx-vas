# User Service – Data Model

Tài liệu này mô tả chi tiết mô hình dữ liệu trong cơ sở dữ liệu của User Service. Bao gồm các bảng: `users`, `roles`, `permissions`, `user_role`, `role_permission`.

```erDiagram
  USERS ||--o{ USER_ROLE : has
  ROLES ||--o{ USER_ROLE : grants
  ROLES ||--o{ ROLE_PERMISSION : defines
  PERMISSIONS ||--o{ ROLE_PERMISSION : belongs
```

---

## 1. `users` – Thông tin người dùng

| Tên cột      | Kiểu dữ liệu     | Ràng buộc                 | Ghi chú                                    |
|--------------|------------------|---------------------------|---------------------------------------------|
| id           | UUID             | PK                        | Định danh duy nhất của người dùng           |
| email        | VARCHAR(255)     | UNIQUE, NOT NULL          | Email đăng nhập                             |
| name         | VARCHAR(255)     |                           | Họ tên người dùng                            |
| status       | VARCHAR(20)      | NOT NULL, DEFAULT 'active'| Trạng thái: `active`, `inactive`            |
| created_at   | TIMESTAMP        | DEFAULT now()             | Thời điểm tạo                               |
| updated_at   | TIMESTAMP        | DEFAULT now()             | Thời điểm cập nhật gần nhất                 |

📌 Một người dùng có thể có nhiều vai trò (quan hệ n-n với bảng `roles`).

---

## 2. `roles` – Vai trò người dùng

| Tên cột      | Kiểu dữ liệu     | Ràng buộc                 | Ghi chú                                    |
|--------------|------------------|---------------------------|---------------------------------------------|
| id           | UUID             | PK                        | Định danh vai trò                           |
| name         | VARCHAR(100)     | UNIQUE, NOT NULL          | Tên vai trò (ví dụ: `teacher`, `admin`)     |
| description  | TEXT             |                           | Mô tả vai trò                                |
| created_at   | TIMESTAMP        | DEFAULT now()             | Thời điểm tạo                               |
| updated_at   | TIMESTAMP        | DEFAULT now()             | Thời điểm cập nhật gần nhất                 |

📌 Một vai trò có thể có nhiều quyền (`permissions`).

---

## 3. `permissions` – Các quyền hệ thống (tĩnh)

| Tên cột      | Kiểu dữ liệu     | Ràng buộc                 | Ghi chú                                      |
|--------------|------------------|---------------------------|-----------------------------------------------|
| id           | UUID             | PK                        | Định danh quyền                               |
| code         | VARCHAR(100)     | UNIQUE, NOT NULL          | Mã quyền: `VIEW_USER_ALL`, `CREATE_USER`...   |
| description  | TEXT             |                           | Mô tả quyền                                   |

📌 Danh sách quyền được migrate tĩnh qua file YAML.

---

## 4. `user_role` – Mapping người dùng ↔ vai trò

| Tên cột   | Kiểu dữ liệu | Ràng buộc                                | Ghi chú                      |
|-----------|--------------|------------------------------------------|-------------------------------|
| user_id   | UUID         | FK → users(id), PK (1/2)                 | ID người dùng                 |
| role_id   | UUID         | FK → roles(id), PK (2/2)                 | ID vai trò được gán          |
| assigned_at | TIMESTAMP  | DEFAULT now()                            | Thời điểm gán vai trò        |

📌 Composite primary key `(user_id, role_id)`

---

## 5. `role_permission` – Mapping vai trò ↔ quyền

| Tên cột       | Kiểu dữ liệu | Ràng buộc                                | Ghi chú                      |
|---------------|--------------|------------------------------------------|-------------------------------|
| role_id       | UUID         | FK → roles(id), PK (1/2)                 | ID vai trò                    |
| permission_id | UUID         | FK → permissions(id), PK (2/2)          | ID quyền                      |
| granted_at    | TIMESTAMP    | DEFAULT now()                            | Thời điểm cấp quyền           |

📌 Composite primary key `(role_id, permission_id)`

---

## 📌 Index & Performance Notes

- `users.email` → unique index.
- Các bảng mapping (`user_role`, `role_permission`) có composite primary key → tự động tạo index.
- Có thể bổ sung index riêng cho các truy vấn thường gặp như:
  - `user_id` trong `user_role`
  - `role_id` trong `role_permission`

---

## 🔐 Các ràng buộc dữ liệu & tính toàn vẹn

- Cascade `ON DELETE` tùy logic nghiệp vụ:
  - `user_role`: Xoá user → xoá bản ghi gán vai trò?
  - `role_permission`: Xoá role → có nên xoá mapping quyền?

  ➜ Quyết định này phụ thuộc vào logic của hệ thống, có thể đặt `ON DELETE CASCADE` hoặc xử lý logic ở tầng ứng dụng.

---

## 🔁 Liên kết tài liệu liên quan

- [Thiết kế tổng thể User Service (design.md)](./design.md)
- [Interface Contract (interface-contract.md)](./interface-contract.md)
- [OpenAPI Spec (openapi.yaml)](./openapi.yaml)

---

