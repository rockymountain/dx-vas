# Kiến trúc Đăng nhập & Phân quyền động (RBAC) – Hệ thống dx_vas

Tài liệu này trình bày chi tiết cách hệ thống dx_vas thực hiện việc xác thực và phân quyền người dùng một cách động (RBAC – Role-Based Access Control) thông qua các thành phần chính như API Gateway, User Service, Auth Service và Redis Cache.

---

## 1. Triết lý thiết kế

Hệ thống dx_vas được thiết kế dựa trên triết lý phân quyền động, đảm bảo mỗi hành động của người dùng trong hệ thống đều được kiểm soát chặt chẽ, linh hoạt và có thể mở rộng theo bối cảnh thực tế của ngành giáo dục.

Các nguyên tắc cốt lõi:

- **RBAC động, dựa trên context (contextual RBAC):** không chỉ dựa vào vai trò tĩnh mà còn đánh giá điều kiện cụ thể trong mỗi request.
- **Condition-Based Access Control:** mọi quyền đều có thể đi kèm với điều kiện được biểu diễn dưới dạng JSONB.
- **Phân tách rõ vai trò và quyền hạn:** vai trò định danh nhiệm vụ, còn permission định nghĩa rõ hành động cụ thể và phạm vi được phép.
- **Không embed permission trong JWT:** permissions được tra cứu realtime từ Redis để đảm bảo tính nhất quán, linh hoạt, và hỗ trợ cập nhật động.
- **Chống đặc quyền vượt mức:** người dùng chỉ được gán role phù hợp, quyền được kiểm soát cấp phát, audit đầy đủ.


---

## 2. Thành phần tham gia

| Thành phần        | Vai trò                                                                 |
|-------------------|--------------------------------------------------------------------------|
| **Người dùng**     | Học sinh, phụ huynh, giáo viên, nhân viên – có thể đăng nhập qua OAuth2/OTP |
| **Frontend App**   | Gửi request kèm JWT tới API Gateway sau khi đăng nhập thành công       |
| **Auth Service**   | Xác thực OAuth2 hoặc OTP → phát hành JWT (`access_token`, `refresh_token`) |
| **API Gateway**    | Điểm kiểm soát trung tâm: xác thực JWT, tra quyền từ Redis, evaluate condition |
| **User Service**   | Cung cấp role, permission và `condition` theo `user_id` qua API/Redis; phát sự kiện khi thay đổi quyền |
| **Redis Cache**    | Lưu danh sách permission đã evaluate – key theo user_id để tăng tốc truy xuất |
| **Backend Services** | Nhận request đã qua kiểm duyệt; chỉ kiểm tra `X-Permissions` header – không cần decode JWT |
| **Audit Logging**  | Lưu toàn bộ hành vi phân quyền, bao gồm gán quyền, kiểm tra RBAC, thay đổi vai trò |

> Tất cả thành phần này được kết nối qua API Gateway – là trung tâm đánh giá bảo mật RBAC động của toàn hệ thống.

---

## 3. Luồng xác thực & phân quyền

Luồng xử lý một request từ người dùng sẽ đi qua các bước:

1. Người dùng đăng nhập qua OAuth2 (GV/NV/HS) hoặc OTP (Phụ huynh)
2. Nhận `access_token` (JWT) từ Auth Service
3. Gửi request tới API Gateway (kèm JWT trong header `Authorization`)
4. API Gateway thực hiện:
   - Xác thực token (decode + verify)
   - Kiểm tra trạng thái `is_active` của user
   - Truy vấn Redis (hoặc fallback DB) để lấy `role`, `permissions`, `condition`
   - Evaluate `condition` theo context hiện tại (VD: student_id trong URL)
   - Nếu hợp lệ → forward request đến backend kèm:
     - `X-User-ID`, `X-Role`, `X-Auth-Method`, `X-Permissions`, `Trace-ID`

5. Backend Service chỉ cần kiểm tra `X-Permissions` để xác nhận quyền.

### 🔄 Sequence Diagram (mô tả logic tổng quát)

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant AuthService
    participant APIGateway
    participant Redis
    participant UserService
    participant Backend

    User->>Frontend: Đăng nhập (OAuth2 / OTP)
    Frontend->>AuthService: Yêu cầu xác thực
    AuthService-->>Frontend: access_token (JWT)
    Frontend->>APIGateway: Gọi API (Authorization: Bearer JWT)

    APIGateway->>AuthService: Xác thực JWT (nội bộ)
    APIGateway->>UserService: Kiểm tra is_active
    APIGateway->>Redis: Lấy role + permission
    alt Cache miss
        APIGateway->>UserService: Truy vấn role/permission từ DB
    end
    APIGateway->>APIGateway: Evaluate condition
    APIGateway->>Backend: Forward request + headers

    Backend->>APIGateway: Xử lý business logic
````

> Lưu ý: Để đơn giản hóa vận hành và tăng hiệu năng, Backend không cần decode JWT hay kiểm tra RBAC lại.

## 4. Cấu trúc dữ liệu RBAC

### Bảng `users`

```sql
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
````

### Bảng `roles`

```sql
CREATE TABLE roles (
  id UUID PRIMARY KEY,
  name TEXT UNIQUE NOT NULL,
  description TEXT
);
```

### Bảng `permissions`

```sql
CREATE TABLE permissions (
  id UUID PRIMARY KEY,
  code TEXT UNIQUE NOT NULL,
  resource TEXT NOT NULL,
  action TEXT NOT NULL,
  condition JSONB -- e.g. { \"student_ids\": [\"abc123\"], \"campus\": \"HCM\" }
);
```

### Bảng ánh xạ

```sql
-- Một user có thể có nhiều role
CREATE TABLE user_role (
  user_id UUID REFERENCES users(id),
  role_id UUID REFERENCES roles(id),
  PRIMARY KEY (user_id, role_id)
);

-- Một role có thể có nhiều permission
CREATE TABLE role_permission (
  role_id UUID REFERENCES roles(id),
  permission_id UUID REFERENCES permissions(id),
  PRIMARY KEY (role_id, permission_id)
);
```

> Các bảng trên sẽ được migrate tĩnh và đồng bộ hóa sang cache Redis để truy xuất nhanh tại Gateway.

---

## 5. Bảo vệ headers định danh

* API Gateway sẽ:

  * Ký các header (`X-Signature`) hoặc
  * Sử dụng Mutual TLS để bảo vệ các service nội bộ
* Các service backend không bao giờ tự tin tưởng header nếu không có lớp bảo vệ này

---

## 6. Tính năng Audit Trail

* Tất cả hành vi quản trị phân quyền được ghi log:

  * Ai thay đổi quyền gì?
  * Khi nào?
  * Trên người dùng nào?
* Gửi log về Cloud Logging → BigQuery để truy vấn, kiểm toán

---

## 7. Công cụ quản trị

* Admin Webapp sẽ có giao diện:

  * Tạo Role
  * Gán Permission
  * Gán Role cho User
  * Thiết lập Condition bằng UI (preset hoặc viết JSON trực tiếp)
  * Xem Audit Log theo user/time/resource

---

## 8. Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)

---

> “Phân quyền đúng – là chìa khóa mở ra đúng cánh cửa, đúng người, đúng thời điểm.”
