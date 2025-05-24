# 📘 Interface Contract – API Gateway (dx\_vas)

## 🧭 Mục tiêu

Tài liệu này mô tả hợp đồng giao tiếp (interface contract) cho **API Gateway** của hệ thống dx\_vas, nhằm chuẩn hóa:

* Định dạng request/response
* Header bắt buộc
* Luồng xác thực và phân quyền (theo ADR-006, ADR-007)
* Hành vi xử lý lỗi và trace (ADR-011, ADR-012)

---

## 🌐 API Groups by Client

| Client          | API Group                                                                                 | Backend Service                                                                 |
|-----------------|-------------------------------------------------------------------------------------------|----------------------------------------------------------------------------------|
| Admin Webapp    | `/users`, `/roles`, `/permissions`                                                       | **User Service**                                                                |
|                 | `/classes`                                                                                | **SIS Adapter**                                                                 |
|                 | `/notifications/send`                                                                     | **Notification Service**                                                        |
| Customer Portal | `/students/me`, `/students/me/scores`, `/students/me/timetable`                          | **LMS Adapter, SIS Adapter**                                                    |
|                 | `/notifications`                                                                          | **Notification Service**                                                        |
| Public Forms    | `/crm/leads` (POST)                                                                       | **CRM Adapter**                                                                 |
| Auth (shared)   | `/auth/*`                                                                                 | **Auth Service**                                                                |

> Mọi gọi API của client đều thông qua Gateway và được định tuyến tới service backend tương ứng, kèm các header `X-User-ID`, `X-Permissions`, `Trace-ID`...

---

## 🔐 Yêu cầu chung cho mọi request

### Header bắt buộc từ client:

| Header          | Mô tả                                                                                |
| --------------- | ------------------------------------------------------------------------------------ |
| `Authorization` | `Bearer <access_token>` – JWT được phát hành từ Auth Service (Google OAuth hoặc OTP) |

### Header bắt buộc được API Gateway forward đến backend:

| Header          | Mô tả                                 | Tham chiếu                                    |
| --------------- | ------------------------------------- | --------------------------------------------- |
| `X-User-ID`     | ID người dùng từ token                | [ADR-006](../ADR/adr-006-auth-strategy.md)    |
| `X-Role`        | Vai trò hiện tại                      | [ADR-006](../ADR/adr-006-auth-strategy.md)    |
| `X-Auth-Method` | `google`, `otp`, `local`              | [ADR-006](../ADR/adr-006-auth-strategy.md)    |
| `X-Permissions` | Danh sách permission code đã evaluate | [ADR-007](../ADR/adr-007-rbac.md)             |
| `Trace-ID`      | Mã trace ID duy nhất theo request     | [ADR-011](../ADR/adr-011-api-error-format.md) |

> Các service backend không cần decode JWT – chỉ sử dụng các header trên.

---

## 📦 Cấu trúc response chuẩn (áp dụng cho mọi API trả về)

```json
{
  "data": {...},
  "error": null,
  "meta": {
    "timestamp": "2025-07-10T12:00:00Z",
    "trace_id": "abc123",
    "source": "api_gateway"
  }
}
```

### Khi lỗi xảy ra:

```json
{
  "data": null,
  "error": {
    "code": "FORBIDDEN",
    "message": "Không đủ quyền truy cập",
    "details": {}
  },
  "meta": {
    "timestamp": "...",
    "trace_id": "abc123"
  }
}
```

> Tuân thủ [ADR-011](../ADR/adr-011-api-error-format.md) và [ADR-012](../ADR/adr-012-response-structure.md).

---

## 🔁 Hành vi xử lý request

| Bước | Diễn giải                                                                     |
| ---- | ----------------------------------------------------------------------------- |
| 1    | API Gateway xác thực JWT hoặc OTP, từ đó lấy `user_id`, `role`, `auth_method` |
| 2    | Kiểm tra `is_active` trong bảng `users` (theo README.md)                      |
| 3    | Tra danh sách permission code qua Redis cache (hoặc fallback DB)              |
| 4    | Kiểm tra permission tương ứng với endpoint đang gọi                           |
| 5    | Gắn header `X-*` và forward đến service đích                                  |

---

## 📌 Lưu ý đặc biệt khi thiết kế endpoint tại backend

* Luôn nhận các header `X-User-ID`, `X-Role`, `X-Permissions`
* **Không cần re-evaluate logic phân quyền**: `X-Permissions` là danh sách permission code đã được Gateway kiểm tra và xác thực theo context hiện tại. Backend chỉ cần kiểm tra code có tồn tại trong danh sách này để cho phép hành động.
* Mọi response đều theo format [ADR-012](../ADR/adr-012-response-structure.md).

---

## 🚨 Các mã lỗi đặc thù của API Gateway (tham khảo)

| Mã lỗi                        | Khi nào xảy ra                              |
| ----------------------------- | ------------------------------------------- |
| `TOKEN_INVALID`               | JWT không hợp lệ hoặc không giải mã được    |
| `TOKEN_EXPIRED`               | JWT đã hết hạn                              |
| `RBAC_CACHE_UNAVAILABLE`      | Redis RBAC cache bị lỗi hoặc không phản hồi |
| `PERMISSION_DENIED`           | Không có permission tương ứng với endpoint  |
| `BACKEND_SERVICE_UNAVAILABLE` | Không gọi được đến service đích             |
| `IS_INACTIVE_USER`            | Người dùng bị khóa (is\_active = false)     |

> Mọi lỗi đều trả về theo cấu trúc chuẩn [ADR-011](../ADR/adr-011-api-error-format.md) với `error.code`, `message`, `details`.

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-011: API Error Format](../ADR/adr-011-api-error-format.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [README.md – Quy trình phân quyền và header forward](../README.md)

---

> “API Gateway không chỉ là người gác cổng – mà là người truyền đi đúng danh tính, đúng quyền hạn, và đúng chuẩn.”
