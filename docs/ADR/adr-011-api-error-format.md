---
id: adr-011-api-error-format
title: ADR-011: Chuẩn hóa định dạng lỗi API cho hệ thống dx_vas
status: accepted
author: DX VAS Architecture Team
date: 2025-06-22
tags: [api, error-handling, design, dx_vas]
---

## 📌 Bối cảnh

Trong hệ thống **dx_vas**, các dịch vụ như API Gateway, LMS Adapter, CRM Adapter, Notification Service... đều cung cấp API cho frontend và/hoặc dịch vụ khác tiêu thụ. Để đảm bảo việc xử lý lỗi dễ dàng và nhất quán:
- Frontend cần phân tích và hiển thị lỗi rõ ràng cho người dùng
- Backend cần log lỗi đầy đủ nhưng không rò rỉ thông tin nhạy cảm
- Các service cần thống nhất định dạng lỗi khi giao tiếp với nhau

Do đó, việc **chuẩn hóa định dạng lỗi API** là cần thiết để hỗ trợ developer, tăng khả năng debug, và đảm bảo tính thống nhất trên toàn hệ thống.

## 🧠 Quyết định

**Áp dụng định dạng lỗi API chuẩn theo JSON, với các trường: `error_code`, `message`, `details`, `trace_id`, `meta`. Tất cả dịch vụ thuộc dx_vas sẽ trả lỗi theo chuẩn này cho mọi HTTP status code từ 4xx trở lên. HTTP status code luôn phản ánh chính xác loại lỗi, và đi kèm với `error_code` tương ứng.**

## 📦 Cấu trúc lỗi chuẩn đề xuất

```json
{
  "error_code": "USER_NOT_FOUND",
  "message": "Người dùng không tồn tại hoặc đã bị xóa",
  "details": {
    "user_id": "u_12345"
  },
  "trace_id": "req-abcd-1234",
  "meta": {
    "timestamp": "2025-06-22T14:00:00Z",
    "service": "gateway",
    "env": "staging"
  }
}
```

### Giải thích các trường:
| Trường | Bắt buộc | Mô tả |
|--------|----------|------|
| `error_code` | ✅ | Mã lỗi tĩnh, viết `SNAKE_CASE`, phục vụ phân loại & xử lý tự động |
| `message` | ✅ | Mô tả lỗi rõ ràng, dùng được để hiển thị cho người dùng cuối nếu cần |
| `details` | ⛔ (optional) | Payload kèm chi tiết lỗi nếu cần debug (không chứa thông tin nhạy cảm) |
| `trace_id` | ⛔ (optional) | Mã định danh request, phục vụ truy vấn log | 
| `meta` | ⛔ (optional) | Thông tin bổ sung: `timestamp`, service name, env, phiên bản API... |

## 🔧 Mapping với HTTP status

| HTTP status | Mô tả | Ví dụ `error_code` |
|-------------|-------|---------------------|
| 400 | Bad Request | `VALIDATION_ERROR`, `MISSING_FIELD` |
| 401 | Unauthorized | `TOKEN_EXPIRED`, `NOT_AUTHENTICATED` |
| 403 | Forbidden | `ACCESS_DENIED`, `RBAC_FORBIDDEN` |
| 404 | Not Found | `USER_NOT_FOUND`, `RESOURCE_NOT_FOUND` |
| 409 | Conflict | `EMAIL_ALREADY_EXISTS` |
| 422 | Unprocessable Entity | `BUSINESS_RULE_VIOLATION` |
| 429 | Too Many Requests | `RATE_LIMIT_EXCEEDED` |
| 500+ | Internal Error | `INTERNAL_ERROR`, `UPSTREAM_FAILURE`, `DB_ERROR` |

## 🔄 Áp dụng toàn hệ thống

- API Gateway: convert mọi exception/error thành response chuẩn
- Các service nội bộ: dùng wrapper/middleware (Python FastAPI, NodeJS Express, Go) để thống nhất định dạng lỗi
- Frontend và mobile chỉ cần bắt lỗi 4xx/5xx và phân tích `error_code`
- Có thể mapping lỗi backend vào frontend enum để hiển thị thân thiện

## 🛠 Tích hợp CI & Linting

- CI/CD scan để đảm bảo mọi endpoint 4xx/5xx đều có JSON `error_code`
- Lint OpenAPI schema để enforce định dạng `{error_code, message, ...}`
- Các error code nên được định nghĩa trong enum riêng (`ERROR_CODES.py`, `error.ts`, ...)

## ✅ Lợi ích

- Frontend dễ hiển thị lỗi và cung cấp gợi ý hành động tiếp theo
- Backend log và phân tích lỗi nhất quán
- Giảm chi phí debug liên dịch vụ
- Dễ thống kê & giám sát lỗi theo `error_code`, `trace_id`

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Service chưa chuyển sang format chuẩn | Cung cấp wrapper/middleware tích hợp dễ dùng |
| Lộ thông tin nhạy cảm qua `details` | Mask dữ liệu + code review chặt chẽ |
| Lỗi trả về sai status code | Kết hợp CI + test + middleware map status chính xác |

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Chỉ trả `message` dạng text | Không phân tích/track được lỗi tự động |
| Dùng HTML/redirect với lỗi | Không phù hợp cho API, mobile, SPA |
| Cho phép mỗi service định nghĩa format riêng | Gây rối loạn, khó maintain & debug |

## 📎 Tài liệu liên quan

- API Governance: [ADR-009](./adr-009-api-governance.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)

---
> “Một hệ thống API chuyên nghiệp không chỉ trả lời đúng – mà còn trả lỗi đúng cách.”
