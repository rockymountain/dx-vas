---
id: adr-011-api-error-format
title: ADR-011: Chuẩn hóa định dạng lỗi API trong hệ thống dx_vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [api, error, format, dx_vas]
---

## 📌 Bối cảnh

Các dịch vụ API trong hệ thống dx_vas phục vụ nhiều loại client: frontend web, mobile app, admin dashboard, service-to-service. Để đảm bảo nhất quán, dễ debug và dễ hiển thị thông báo lỗi cho người dùng, cần một định dạng lỗi chuẩn hóa trên toàn hệ thống.

> 🔄 Quyết định sử dụng `trace_id` (thay vì `request_id`) để đồng bộ với hệ thống quan sát phân tán (tracing), thống nhất với ADR-005 (Observability) và ADR-008 (Audit Logging).

---

## 🧠 Quyết định

Tất cả lỗi API sẽ trả về response JSON thống nhất theo cấu trúc:
```json
{
  "error": {
    "code": "<ERROR_CODE>",
    "message": "<Human-readable message>",
    "details": {
      // optional, context phụ trợ cho dev/debug
    }
  },
  "meta": {
    "timestamp": "2025-06-22T12:34:56Z",
    "trace_id": "trace-12345"
  }
}
```
- HTTP status code vẫn trả về đúng chuẩn REST (`400`, `401`, `403`, `404`, `500`...)
- `error.code`: mã lỗi tĩnh, để frontend hoặc hệ thống khác dễ bắt & dịch
- `error.message`: mô tả dễ hiểu với người dùng
- `error.details`: có thể bao gồm field cụ thể sai, hoặc context thêm
- `meta.trace_id`: ID duy nhất cho mỗi request dùng trong trace/log phân tán

---

## 🧾 Danh sách mã lỗi chuẩn hóa

| error.code | HTTP code | Ý nghĩa |
|------------|-----------|---------|
| `INVALID_INPUT` | 400 | Dữ liệu đầu vào không hợp lệ |
| `UNAUTHORIZED` | 401 | Chưa đăng nhập hoặc token sai |
| `FORBIDDEN` | 403 | Không đủ quyền |
| `NOT_FOUND` | 404 | Tài nguyên không tồn tại |
| `INTERNAL_ERROR` | 500 | Lỗi hệ thống nội bộ |
| `VALIDATION_FAILED` | 422 | Dữ liệu hợp lệ về schema nhưng sai logic |
| `OTP_EXPIRED` | 400 | Mã OTP đã hết hạn (dùng trong login phụ huynh) |
| `OTP_INVALID` | 400 | Mã OTP sai hoặc không khớp |
| `RATE_LIMITED` | 429 | Gọi API quá nhiều trong thời gian ngắn |

---

## 📦 Mẫu phản hồi chi tiết

### ❌ Trường hợp lỗi (HTTP 400 – OTP hết hạn)
```json
{
  "error": {
    "code": "OTP_EXPIRED",
    "message": "Mã xác nhận đã hết hạn. Vui lòng thử lại."
  },
  "meta": {
    "timestamp": "2025-06-22T14:01:02Z",
    "trace_id": "trace-otp-abcd123"
  }
}
```

### ✅ Trường hợp thành công
```json
{
  "data": {
    "user": {
      "id": "u123",
      "role": "parent"
    },
    "token": "<JWT token>"
  },
  "meta": {
    "timestamp": "2025-06-22T14:01:02Z",
    "trace_id": "trace-otp-abcd123"
  }
}
```

---

## 🧰 Định nghĩa OpenAPI Schema (áp dụng từ ADR-012)
- `ApiErrorInternal`: phần bên trong `error`
```yaml
ApiErrorInternal:
  type: object
  required:
    - code
    - message
  properties:
    code:
      type: string
      example: OTP_EXPIRED
    message:
      type: string
      example: Mã xác nhận đã hết hạn
    details:
      type: object
      additionalProperties: true
```
- `ApiErrorResponse`:
```yaml
ApiErrorResponse:
  type: object
  properties:
    error:
      $ref: '#/components/schemas/ApiErrorInternal'
    meta:
      $ref: '#/components/schemas/ResponseMeta'
```
- `ResponseMeta`:
```yaml
ResponseMeta:
  type: object
  properties:
    timestamp:
      type: string
      format: date-time
    trace_id:
      type: string
```

---

## ✅ Lợi ích
- Giao tiếp API nhất quán, dễ xử lý ở client
- Giúp debug nhanh thông qua `trace_id`
- Frontend dễ hiển thị thông báo lỗi có ngữ nghĩa

---

## ❌ Rủi ro & Giải pháp
| Rủi ro | Giải pháp |
|--------|-----------|
| Quên không dùng chuẩn | CI rule check, middleware kiểm chuẩn |
| Lỗi ẩn dưới 200 OK | Phải throw rõ ràng trong controller/service |

---

## 📎 Tài liệu liên quan
- Observability: [ADR-005](./adr-005-observability.md)
- Auth Strategy: [ADR-006](./adr-006-auth-strategy.md)
- RBAC & Permission: [ADR-007](./adr-007-rbac.md)
- API Governance: [ADR-009](./adr-009-api-governance.md)
- API Response Structure: [ADR-012](./adr-012-response-structure.md)
- Audit Logging: [ADR-008](./adr-008-audit-logging.md)

---
> “Lỗi là điều không thể tránh — nhưng chuẩn hóa giúp ta phản hồi đúng và kịp thời.”
