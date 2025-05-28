---
id: adr-012-response-structure
title: ADR-012 - Chuẩn hóa cấu trúc phản hồi (API Response Structure) cho hệ thống dx-vas
status: accepted
author: DX VAS Architecture Team
date: 2025-05-22
tags: [api, design, response, standard, dx-vas]
---

## 📌 Bối cảnh

Các API trong hệ thống **dx-vas** cần đảm bảo:

* Frontend có thể xử lý response dễ dàng và đồng nhất
* Các service có thể tích hợp với nhau với ít công sức chuyển đổi định dạng
* Dữ liệu, lỗi, và thông tin phụ trợ (metadata) được tách bạch rõ ràng

Hiện tại, một số API trả về:

* Dữ liệu gốc (list, object)
* Dữ liệu gói trong `{data: ...}`
* Một số có thêm `{meta}`, số khác thì không

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

* `data`: nội dung chính mà API trả về (có thể là object, list, null)
* `error`: null nếu thành công, hoặc object lỗi **tuân thủ hoàn toàn theo ADR-011**, gồm `code`, `message`, `details`
* `meta`: thông tin phụ trợ của response, **luôn bao gồm `timestamp` và `trace_id`** như trong ADR-011, có thể mở rộng thêm như `pagination`, `version`, `source`, `duration_ms`...

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
    "trace_id": "trace-success-xyz789",
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
    "code": "STUDENT_NOT_FOUND",
    "message": "Không tìm thấy học sinh với ID s_999",
    "details": {
      "student_id": "s_999"
    }
  },
  "meta": {
    "timestamp": "2025-06-22T12:01:00Z",
    "trace_id": "abc-1234",
    "source": "lms_proxy"
  }
}
```

---

## 🔧 Nguyên tắc áp dụng

* **Luôn có đủ 3 trường `data`, `error`, `meta`** trong mọi response
* **Nếu lỗi xảy ra**, `data = null`, `error` chứa thông tin lỗi chuẩn → tham chiếu [`adr-011`](./adr-011-api-error-format.md)
* **Nếu thành công**, `error = null`
* **`meta` luôn bao gồm `timestamp`, `trace_id`**, và có thể mở rộng thêm các trường như: `pagination`, `version`, `source`, `duration_ms`

---

## 🛠 Tích hợp CI & OpenAPI

```yaml
ApiResponse:
  type: object
  required:
    - data
    - error
    - meta
  properties:
    data: {}
    error:
      type: object
      nullable: true
      required:
        - code
        - message
      properties:
        code:
          type: string
          description: Mã lỗi tĩnh.
        message:
          type: string
          description: Mô tả lỗi dành cho người dùng.
        details:
          type: object
          nullable: true
          description: Thông tin chi tiết bổ sung cho việc gỡ lỗi (tùy chọn).
          additionalProperties: true
    meta:
      $ref: '#/components/schemas/ResponseMeta'

ResponseMeta:
  type: object
  required:
    - timestamp
    - trace_id
  properties:
    timestamp:
      type: string
      format: date-time
      description: Thời điểm tạo response.
    trace_id:
      type: string
      description: ID duy nhất để theo dõi request qua các hệ thống.
    source:
      type: string
      description: Tên service tạo ra response.
    version:
      type: string
      description: Phiên bản của API.
    duration_ms:
      type: integer
      description: Thời gian xử lý request (nếu có)
```

* Check trong CI linter: endpoint nào không trả về đủ 3 field sẽ bị cảnh báo
* Mẫu response chuẩn được render trong Swagger / Redoc cho tất cả dịch vụ

---

## ✅ Lợi ích

* Frontend dễ dàng xây dựng handler cho response chung
* Dễ log, thống kê, phân tích request/response toàn hệ thống
* Giao tiếp service-to-service rõ ràng, dễ test và mock
* Đồng bộ với error handling (`adr-011`) và API Governance (`adr-009`)

---

## ❌ Rủi ro & Giải pháp

| Rủi ro                                                  | Giải pháp                                                             |
| ------------------------------------------------------- | --------------------------------------------------------------------- |
| Một số service legacy chưa chuyển đổi                   | Cung cấp wrapper chuẩn để hỗ trợ nhanh chuyển đổi                     |
| Dữ liệu quá lớn dẫn đến response nặng                   | Sử dụng paging, limit, lazy loading hoặc exclude `meta` nếu không cần |
| Frontend quên kiểm tra `error` và dùng trực tiếp `data` | Lint hoặc wrapper ở client để enforce quy tắc xử lý response          |

---

## 🔄 Các phương án đã loại bỏ

| Phương án                       | Lý do không chọn                       |
| ------------------------------- | -------------------------------------- |
| Trả trực tiếp object hoặc list  | Không phân biệt rõ thành công/thất bại |
| Trả riêng cấu trúc cho từng API | Gây rối loạn, khó bảo trì và tích hợp  |

---

## 📎 Tài liệu liên quan

* API Error format: [ADR-011](./adr-011-api-error-format.md)
* API Governance: [ADR-009](./adr-009-api-governance.md)
* CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)

---

> "API không chỉ cần đúng dữ liệu – mà còn cần đúng định dạng."
