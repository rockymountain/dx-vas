# 📘 Service Interface Contracts – API Gateway & Admin Webapp (dx\_vas)

## 🧭 Mục tiêu

Chuẩn hóa hợp đồng giao tiếp (interface contract) giữa hai service quan trọng:

* API Gateway (trung tâm kiểm soát xác thực, phân quyền, định tuyến request)
* Admin Webapp (ứng dụng quản trị toàn hệ thống)

---

## 1️⃣ API Gateway

### 🎯 Vai trò

* Xác thực (OAuth2, Local, OTP)
* Áp dụng phân quyền RBAC động (theo ADR-007)
* Forward request đến các service phía sau (CRM, LMS, SIS...)

### 🔐 Header chuẩn forward

| Header          | Ý nghĩa                                                                     |
| --------------- | --------------------------------------------------------------------------- |
| `X-User-ID`     | ID người dùng đã xác thực                                                   |
| `X-Role`        | Vai trò hiện hành của người dùng                                            |
| `X-Auth-Method` | `google`, `otp`, `local`                                                    |
| `X-Permissions` | Danh sách các `permission_code` hợp lệ (đã evaluate tại Gateway)            |
| `Trace-ID`      | ID duy nhất cho mỗi request (theo [ADR-011](./adr-011-api-error-format.md)) |

### 📥 Input

* Request từ client (Admin Webapp, Customer Portal...) với JWT hoặc OTP token

### 📤 Output

* Forward tới backend: CRM, LMS, Notification, ... kèm headers như trên

### 🔎 Xử lý nội bộ

* Tra cứu quyền từ Redis (RBAC Cache)
* Kiểm tra điều kiện `is_active` (README.md)
* Kiểm tra `permission` theo pattern: resource + action + condition (ADR-007)

---

## 2️⃣ Admin Webapp

### 🎯 Vai trò

* Giao diện quản trị tập trung: User, Role, Permission, Class, Fee, Audit
* Kết nối API Gateway để thực hiện thao tác đọc/ghi tới backend services

### 🔐 Yêu cầu truy cập

* Luôn gửi JWT khi gọi API Gateway
* Phải nằm trong vai trò có đủ permission tương ứng với endpoint yêu cầu

### 🔧 Endpoint tiêu biểu (gọi tới API Gateway)

| Method | Endpoint              | Ý nghĩa                       | Permission Code         | Input Schema / Query Params      | Output Schema                |
| ------ | --------------------- | ----------------------------- | ----------------------- | -------------------------------- | ---------------------------- |
| GET    | `/users`              | Xem danh sách người dùng      | `VIEW_USER_ALL`         | `?page`, `?search`               | `List[UserOut]`              |
| POST   | `/users`              | Tạo người dùng mới            | `CREATE_USER`           | `UserCreate` (JSON body)         | `UserOut`                    |
| PATCH  | `/users/{id}`         | Cập nhật thông tin người dùng | `EDIT_USER`             | `UserUpdate` (JSON body)         | `UserOut`                    |
| GET    | `/permissions`        | Lấy danh sách permission      | `VIEW_PERMISSION_ALL`   | —                                | `List[PermissionOut]`        |
| POST   | `/roles`              | Tạo mới role                  | `CREATE_ROLE`           | `RoleCreate`                     | `RoleOut`                    |
| GET    | `/audit/logs`         | Xem lịch sử thay đổi dữ liệu  | `VIEW_AUDIT_LOG`        | `?resource`, `?user_id`, `?date` | `List[AuditLogOut]`          |
| POST   | `/classes`            | Tạo lớp học                   | `CREATE_CLASS`          | `ClassCreate`                    | `ClassOut`                   |
| POST   | `/notifications/send` | Gửi thông báo tới người dùng  | `SEND_NOTIFICATION_ALL` | `NotificationRequest`            | `NotificationDispatchResult` |

> Các schema có thể được định nghĩa bằng Pydantic hoặc JSON Schema tương đương. Được dùng bởi cả backend và frontend để validate.

### 📤 Phản hồi từ API Gateway

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

### 🚨 Phản hồi lỗi (tuân thủ ADR-011)

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

## ✅ Ghi chú chung

* Mọi request đều phải được trace bằng `trace_id` từ đầu đến cuối chuỗi dịch vụ (xem ADR-011)
* Các `permission_code` được API Gateway evaluate và forward là **duy nhất**, **được backend tin cậy**, và **là cơ sở để quyết định cho phép hay từ chối** hành động tương ứng.
* Response structure của tất cả các service phải tuân theo ADR-012 để frontend có thể xử lý thống nhất.

---

> “Interface contract rõ ràng là bước đầu tiên để mỗi service có thể phát triển độc lập nhưng phối hợp hoàn hảo.”
