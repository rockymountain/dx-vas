# User Service – Data Model

Tài liệu này mô tả chi tiết các bảng dữ liệu chính trong hệ thống `User Service`, được thiết kế để phục vụ cho:

- Quản lý thông tin người dùng (User)
- Quản lý vai trò, quyền hạn (RBAC)
- Quản lý lịch sử, trạng thái người dùng

---

## Mục lục

1. [users](#users)
2. [roles](#roles)
3. [permissions](#permissions)
4. [user_roles](#user_roles)
5. [role_permissions](#role_permissions)
6. [processed_messages (idempotency)](#processed_messages-idempotency)

---

## users

| Tên cột       | Kiểu dữ liệu | Ràng buộc         | Mô tả                         |
|---------------|--------------|--------------------|-------------------------------|
| id            | UUID         | PK                 | Mã định danh người dùng       |
| email         | TEXT         | UNIQUE, NOT NULL   | Email duy nhất                |
| full_name     | TEXT         |                    | Tên đầy đủ                    |
| is_active     | BOOLEAN      | DEFAULT TRUE       | Trạng thái hoạt động          |
| created_at    | TIMESTAMP    | DEFAULT now()      | Thời điểm tạo                 |
| updated_at    | TIMESTAMP    | DEFAULT now()      | Thời điểm cập nhật gần nhất   |

---

## roles

| Tên cột     | Kiểu dữ liệu | Ràng buộc       | Mô tả                      |
|-------------|--------------|------------------|----------------------------|
| id          | UUID         | PK               | Mã định danh vai trò       |
| code        | TEXT         | UNIQUE, NOT NULL | Mã vai trò (vd: PARENT)    |
| name        | TEXT         |                  | Tên hiển thị của vai trò   |

---

## permissions

| Tên cột     | Kiểu dữ liệu | Ràng buộc       | Mô tả                            |
|-------------|--------------|------------------|----------------------------------|
| id          | UUID         | PK               | Mã định danh quyền               |
| code        | TEXT         | UNIQUE, NOT NULL | Mã quyền (vd: VIEW_SCORE_OWN_CHILD) |
| resource    | TEXT         | NOT NULL         | Tài nguyên (vd: student_score)   |
| action      | TEXT         | NOT NULL         | Hành động (vd: view, edit)       |
| condition   | JSONB        |                  | Điều kiện áp dụng (có thể null)  |

---

## user_roles

| Tên cột     | Kiểu dữ liệu | Ràng buộc                 | Mô tả                    |
|-------------|--------------|----------------------------|--------------------------|
| user_id     | UUID         | FK → users(id), PK         | Người dùng               |
| role_id     | UUID         | FK → roles(id), PK         | Vai trò của người dùng   |

---

## role_permissions

| Tên cột     | Kiểu dữ liệu | Ràng buộc                     | Mô tả                         |
|-------------|--------------|-------------------------------|-------------------------------|
| role_id     | UUID         | FK → roles(id), PK            | Vai trò                       |
| permission_id | UUID       | FK → permissions(id), PK      | Quyền thuộc vai trò đó        |

---

## processed_messages (idempotency)

| Tên cột        | Kiểu dữ liệu | Ràng buộc         | Mô tả                                    |
|----------------|--------------|--------------------|------------------------------------------|
| message_id     | TEXT         | PK                 | ID thông điệp đã xử lý                   |
| processed_at   | TIMESTAMP    | DEFAULT now()      | Thời điểm xử lý                         |
| status         | TEXT         |                    | Trạng thái xử lý (ví dụ: success, fail) |

---

📌 **Lưu ý:**
- Tất cả các UUID nên sử dụng chuẩn `uuid_generate_v4()` (PostgreSQL).
- Các timestamp sử dụng `DEFAULT now()` và được cập nhật tự động bằng trigger nếu có.
- Tài liệu này có thể cập nhật song song với file migration Alembic.

