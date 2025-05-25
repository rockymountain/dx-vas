# Kiến trúc Đăng nhập & Phân quyền động (RBAC) – Hệ thống dx_vas

Tài liệu này trình bày chi tiết cách hệ thống dx_vas thực hiện việc xác thực và phân quyền người dùng một cách động (RBAC – Role-Based Access Control) thông qua các thành phần chính như API Gateway, User Service, Auth Service và Redis Cache.

---

## 1. Triết lý thiết kế

- **RBAC động:** Các quyền được gán dựa trên vai trò nhưng có thể kèm điều kiện theo ngữ cảnh.
- **Không nhúng permission vào JWT:** Tránh stale data, đảm bảo luôn lấy thông tin quyền mới nhất từ Redis/DB.
- **Điều kiện chi tiết:** Cho phép giới hạn quyền dựa theo `student_id`, `class_id`, `campus`, v.v.

---

## 2. Thành phần tham gia

| Thành phần        | Vai trò                                                                 |
|-------------------|--------------------------------------------------------------------------|
| **Auth Service**  | Phát hành JWT sau xác thực OAuth2/OTP                                   |
| **API Gateway**   | Kiểm tra JWT, tra Redis để lấy role/permission, kiểm tra `is_active`    |
| **User Service**  | Quản lý user, role, permission, trạng thái tài khoản, audit             |
| **Redis Cache**   | Lưu danh sách permission đã evaluate để tăng hiệu năng                 |
| **Admin Webapp**  | Giao diện quản lý user, role, permission, điều kiện phân quyền         |

---

## 3. Luồng xác thực & phân quyền

1. Người dùng đăng nhập qua Google OAuth2 (GV/NV/HS) hoặc OTP (Phụ huynh)
2. Nhận `access_token` từ Auth Service (không chứa permission)
3. Gọi API → Gateway:
   - Xác thực JWT
   - Kiểm tra `is_active` từ bảng `users`
   - Tra role và permission từ Redis (hoặc DB nếu cache miss)
   - Evaluate điều kiện (condition) → tạo danh sách permission code
   - Gửi request kèm các header:
     - `X-User-ID`
     - `X-Role`
     - `X-Permissions` (danh sách code)
     - `Trace-ID`

> Backend không cần decode JWT hay tính lại RBAC. Chỉ cần kiểm tra `X-Permissions`.

---

## 4. Cấu trúc dữ liệu RBAC

```sql
-- users table
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  auth_provider TEXT NOT NULL DEFAULT 'google',
  user_category TEXT NOT NULL CHECK (user_category IN ('student', 'teacher', 'staff', 'parent')),
  password_hash TEXT,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now()
);

-- roles
CREATE TABLE roles (
  id UUID PRIMARY KEY,
  name TEXT UNIQUE NOT NULL,
  description TEXT
);

-- permissions
CREATE TABLE permissions (
  id UUID PRIMARY KEY,
  code TEXT UNIQUE NOT NULL,
  resource TEXT NOT NULL,
  action TEXT NOT NULL,
  condition JSONB  -- e.g. { \"class_id\": \"...\", \"campus\": \"HCM\" }
);

-- mapping
user_role (user_id, role_id)
role_permission (role_id, permission_id)

---

## 5. Ví dụ về permission có điều kiện

```json
{
  "code": "VIEW_STUDENT_SCORE_OWN_CHILD",
  "resource": "student_score",
  "action": "view",
  "condition": {
    "accessible_student_ids": ["123e4567-e89b-12d3-a456-426614174000"]
  }
}
```

> Điều kiện được evaluate tại API Gateway để xác định người dùng chỉ có thể truy cập học sinh là con mình.

---

## 6. Bảo vệ headers định danh

* API Gateway sẽ:

  * Ký các header (`X-Signature`) hoặc
  * Sử dụng Mutual TLS để bảo vệ các service nội bộ
* Các service backend không bao giờ tự tin tưởng header nếu không có lớp bảo vệ này

---

## 7. Tính năng Audit Trail

* Tất cả hành vi quản trị phân quyền được ghi log:

  * Ai thay đổi quyền gì?
  * Khi nào?
  * Trên người dùng nào?
* Gửi log về Cloud Logging → BigQuery để truy vấn, kiểm toán

---

## 8. Công cụ quản trị

* Admin Webapp sẽ có giao diện:

  * Tạo Role
  * Gán Permission
  * Gán Role cho User
  * Thiết lập Condition bằng UI (preset hoặc viết JSON trực tiếp)
  * Xem Audit Log theo user/time/resource

---

## 9. Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)

---

> “Phân quyền đúng – là chìa khóa mở ra đúng cánh cửa, đúng người, đúng thời điểm.”
