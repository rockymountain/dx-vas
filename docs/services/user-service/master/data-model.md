# 🗄️ User Service Master – Mô hình Dữ liệu (Data Model)

---

## 1. Bảng `users_global`

| Cột | Kiểu dữ liệu | Ràng buộc | Ghi chú |
|------|--------------|------------|--------|
| user_id | UUID | PRIMARY KEY | Định danh toàn cục |
| full_name | TEXT | NOT NULL | Tên đầy đủ |
| email | TEXT | NOT NULL, UNIQUE | Bắt buộc nếu dùng Google login |
| phone | TEXT |  | Có thể null |
| auth_provider | TEXT | NOT NULL | `google`, `otp`, `local`, ... |
| created_at | TIMESTAMP | DEFAULT now() | Thời điểm tạo |

---

## 2. Bảng `tenants`

| Cột | Kiểu dữ liệu | Ràng buộc | Ghi chú |
|------|--------------|------------|--------|
| tenant_id | TEXT | PRIMARY KEY | Ví dụ: `tenant_a`, `tenant_b` |
| tenant_name | TEXT | NOT NULL | Tên hiển thị |
| status | TEXT | DEFAULT 'active' | `active` / `inactive` |
| created_at | TIMESTAMP | DEFAULT now() | |

---

## 3. Bảng `user_tenant_assignments`

| Cột | Kiểu dữ liệu | Ràng buộc | Ghi chú |
|------|--------------|------------|--------|
| assignment_id | UUID | PRIMARY KEY | |
| user_id | UUID | FK → `users_global(user_id)` | |
| tenant_id | TEXT | FK → `tenants(tenant_id)` | |
| role_codes | TEXT[] | NOT NULL | Danh sách `role_code` người dùng có tại tenant |
| is_active | BOOLEAN | DEFAULT true | Trạng thái trong tenant |
| created_at | TIMESTAMP | DEFAULT now() | |

---

## 4. Bảng `global_roles_templates`

| Cột | Kiểu dữ liệu | Ràng buộc | Ghi chú |
|------|--------------|------------|--------|
| role_template_id | UUID | PRIMARY KEY | |
| role_code | TEXT | UNIQUE, NOT NULL | Mã vai trò – dùng trong RBAC |
| role_name | TEXT | NOT NULL | Tên hiển thị |
| description | TEXT |  | |

---

## 5. Bảng `global_permissions_templates`

| Cột | Kiểu dữ liệu | Ràng buộc | Ghi chú |
|------|--------------|------------|--------|
| permission_template_id | UUID | PRIMARY KEY | |
| permission_code | TEXT | UNIQUE, NOT NULL | Mã quyền – dùng trong RBAC |
| action | TEXT | NOT NULL | Ví dụ: `read`, `update`, `assign_role` |
| resource | TEXT | NOT NULL | Ví dụ: `user`, `classroom`, `notification` |
| default_condition | JSONB |  | Điều kiện mặc định áp dụng |
| description | TEXT |  | |

---

## 🔁 Quan hệ và Logic tổng quan

- Một `user_id` có thể được gán vào nhiều tenant qua `user_tenant_assignments`.
- Một tenant có thể có nhiều người dùng, mỗi người dùng có nhiều `role_codes`.
- Các `role_codes` & `permission_codes` được chuẩn hóa từ template toàn cục, dùng để seed xuống Sub User Service.
- Sub User Service sẽ ánh xạ theo `user_id_global`.

---

## 🧪 Gợi ý kiểm thử dữ liệu

```sql
-- Thêm user toàn cục
INSERT INTO users_global (user_id, full_name, email, auth_provider)
VALUES (gen_random_uuid(), 'Nguyễn Văn A', 'a@vietanh.edu.vn', 'google');

-- Thêm tenant
INSERT INTO tenants (tenant_id, tenant_name) VALUES ('tenant_a', 'Trường Việt Anh A');

-- Gán user vào tenant với vai trò teacher + librarian
INSERT INTO user_tenant_assignments (assignment_id, user_id, tenant_id, role_codes)
VALUES (gen_random_uuid(), '...', 'tenant_a', ARRAY['teacher', 'librarian']);
```

---

📎 Tài liệu liên quan:

* [Thiết kế tổng quan (design.md)](./design.md)
* [RBAC Deep Dive](../../../rbac-deep-dive.md)
* [ADR-007: RBAC Strategy](../../../adr-007-rbac.md)
