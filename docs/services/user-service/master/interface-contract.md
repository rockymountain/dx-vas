# 📘 User Service Master – Interface Contract

Tài liệu này mô tả các API chính mà User Service Master cung cấp, theo phong cách dễ đọc cho developer/backend/frontend. Đặc tả kỹ thuật chi tiết xem thêm tại [`openapi.yaml`](./openapi.yaml).

---

## 📂 Nhóm: Users Global

### 🔹 `GET /users-global/by-email`

**Mô tả:**  
Truy vấn người dùng toàn cục theo email.

**Query Params:**
- `email`: địa chỉ email (bắt buộc)

**Response:**
```json
{
  "user_id": "uuid",
  "full_name": "Nguyễn Văn A",
  "email": "abc@example.com",
  "auth_provider": "google",
  "created_at": "2024-10-01T10:00:00Z"
}
```

---

### 🔹 `POST /users-global`

**Mô tả:**
Tạo mới người dùng toàn cục (dành cho Auth Master khi user lần đầu login bằng Google).

**Request Body:**

```json
{
  "full_name": "Nguyễn Văn A",
  "email": "abc@example.com",
  "auth_provider": "google"
}
```

**Response:**

```json
{
  "user_id": "uuid"
}
```

---

## 📂 Nhóm: Tenants

### 🔹 `GET /tenants`

**Mô tả:**
Lấy danh sách tất cả các tenant hiện có trong hệ thống.

**Response:**

```json
[
  {
    "tenant_id": "tenant_a",
    "tenant_name": "Trường Việt Anh A",
    "status": "active"
  },
  {
    "tenant_id": "tenant_b",
    "tenant_name": "Trường Việt Anh B",
    "status": "inactive"
  }
]
```

---

## 📂 Nhóm: User – Tenant Assignment

### 🔹 `GET /user-tenant-assignments?user_id=...`

**Mô tả:**
Trả về danh sách tenant mà một người dùng thuộc về.

**Response:**

```json
[
  {
    "tenant_id": "tenant_a",
    "role_codes": ["teacher", "homeroom"],
    "is_active": true
  },
  ...
]
```

---

### 🔹 `POST /user-tenant-assignments`

**Mô tả:**
Gán người dùng vào một tenant, với các vai trò cụ thể.

**Request Body:**

```json
{
  "user_id": "uuid",
  "tenant_id": "tenant_a",
  "role_codes": ["teacher", "librarian"]
}
```

**Response:**

```json
{
  "status": "success"
}
```

---

## 📝 Ghi chú:

* Các API này chỉ được gọi bởi Auth Service, Superadmin Webapp hoặc Sub Services (nếu có token kỹ thuật).
* Các lỗi có thể trả về theo chuẩn [`adr-011-api-error-format.md`](../../../adr-011-api-error-format.md).
* Mọi thay đổi định danh người dùng cần được ghi log audit (xem [`adr-008-audit-logging.md`](../../../adr-008-audit-logging.md)).

---

📎 Để biết chi tiết luồng nghiệp vụ: xem [`design.md`](./design.md)
📦 Để tra cứu schema chi tiết: xem [`data-model.md`](./data-model.md)
