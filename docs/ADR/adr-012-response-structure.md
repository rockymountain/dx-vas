---
id: adr-012-response-structure
title: ADR-012: Chuẩn hóa cấu trúc phản hồi (API Response Structure) cho hệ thống dx_vas
status: accepted
author: DX VAS Architecture Team
date: 2025-06-22
tags: [api, design, response, standard, dx_vas]
---

## 📌 Bối cảnh

Các API trong hệ thống **dx_vas** cần đảm bảo:
- Frontend có thể xử lý response dễ dàng và đồng nhất
- Các service có thể tích hợp với nhau với ít công sức chuyển đổi định dạng
- Dữ liệu, lỗi, và thông tin phụ trợ (metadata) được tách bạch rõ ràng

Hiện tại, một số API trả về:
- Dữ liệu gốc (list, object)
- Dữ liệu gói trong `{data: ...}`
- Một số có thêm `{meta}`, số khác thì không

Điều này gây khó khăn khi thống nhất và tích hợp đa hệ thống (API Gateway, CRM Adapter, LMS Proxy, Notification Service...).

---

## 🧠 Quyết định

**Chuẩn hóa tất cả API trả về cấu trúc phản hồi dạng JSON:**
```json
{
  "data": {...},
  "error": null,
  "meta": {...}
}
```

- `data`: nội dung chính mà API trả về (có thể là object, list, null)
- `error`: null nếu thành công, hoặc object theo chuẩn [`adr-011-api-error-format.md`](./adr-011-api-error-format.md) nếu có lỗi
- `meta`: thông tin phụ trợ như phân trang, tổng số bản ghi, version, timestamp...

---

## 📦 Mẫu phản hồi chi tiết

### ✅ Trường hợp thành công (HTTP 200)
```json
{
  "data": {
    "student_id": "s_123",
    "name": "Nguyễn Văn A",
    "class": "5A"
  },
  "error": null,
  "meta": {
    "timestamp": "2025-06-22T12:00:00Z",
    "version": "v1",
    "source": "lms_proxy"
  }
}
```

### ❌ Trường hợp lỗi (HTTP 404)
```json
{
  "data": null,
  "error": {
    "error_code": "STUDENT_NOT_FOUND",
    "message": "Không tìm thấy học sinh với ID s_999",
    "details": {
      "student_id": "s_999"
    },
    "trace_id": "abc-1234",
    "meta": {
      "timestamp": "2025-06-22T12:01:00Z",
      "service": "lms_proxy",
      "env": "staging"
    }
  },
  "meta": {
    "timestamp": "2025-06-22T12:01:00Z",
    "source": "lms_proxy"
  }
}
```

---

## 🔧 Nguyên tắc áp dụng

- **Luôn có đủ 3 trường `data`, `error`, `meta`** trong mọi response
- **Nếu lỗi xảy ra**, `data = null`, `error` chứa thông tin lỗi chuẩn → tham chiếu [`adr-011`](./adr-011-api-error-format.md)
- **Nếu thành công**, `error = null`
- `meta` có thể mở rộng thêm thông tin: `pagination`, `request_id`, `service`, `env`, `duration_ms`, ...

---

## 🛠 Tích hợp CI & OpenAPI

- OpenAPI schema chuẩn cho tất cả response:
```yaml
ApiResponse:
  type: object
  required: [data, error, meta]
  properties:
    data:
      type: object
    error:
      $ref: '#/components/schemas/ApiError'
    meta:
      type: object

ApiError:
  type: object
  required: [error_code, message]
  properties:
    error_code:
      type: string
      example: STUDENT_NOT_FOUND
    message:
      type: string
      example: Không tìm thấy học sinh với ID s_999
    details:
      type: object
      description: Thông tin bổ sung để debug lỗi
    trace_id:
      type: string
      example: abc-1234
    meta:
      type: object
      properties:
        timestamp:
          type: string
          format: date-time
        service:
          type: string
        env:
          type: string
```

- Check trong linter: endpoint nào không trả về đủ 3 field sẽ bị cảnh báo
- Mẫu response chuẩn được render trong Swagger / Redoc cho tất cả dịch vụ

---

## ✅ Lợi ích

- Frontend dễ dàng xây dựng handler cho response chung
- Dễ log, thống kê, phân tích request/response toàn hệ thống
- Giao tiếp service-to-service rõ ràng, dễ test và mock
- Đồng bộ với error handling (`adr-011`) và API Governance (`adr-009`)

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Một số service legacy chưa chuyển đổi | Cung cấp wrapper chuẩn để hỗ trợ nhanh chuyển đổi |
| Dữ liệu quá lớn dẫn đến response nặng | Sử dụng paging, limit, lazy loading hoặc exclude `meta` nếu không cần |
| Frontend quên kiểm tra `error` và dùng trực tiếp `data` | Lint hoặc wrapper ở client để enforce quy tắc xử lý response |

---

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Trả trực tiếp object hoặc list | Không phân biệt rõ thành công/thất bại |
| Trả riêng cấu trúc cho từng API | Gây rối loạn, khó bảo trì và tích hợp |

---

## 📎 Tài liệu liên quan

- API Error format: [ADR-011](./adr-011-api-error-format.md)
- API Governance: [ADR-009](./adr-009-api-governance.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)

---
> "API không chỉ cần đúng dữ liệu – mà còn cần đúng định dạng."
