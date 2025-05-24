# 📘 Interface Contract – Notification Service (dx\_vas)

## 🧭 Mục tiêu

Tài liệu này định nghĩa interface contract cho **Notification Service**, thành phần chịu trách nhiệm:

* Gửi thông báo qua email, SMS, app, và Zalo
* Nhận yêu cầu từ các hệ thống khác qua API Gateway
* Tự động hoá các luồng gửi theo template và định tuyến kênh gửi phù hợp

---

## 🔁 Các cách Notification Service được gọi (Consumer)

| Source Service | Method | Endpoint                           | Mô tả                                                                     |
| -------------- | ------ | ---------------------------------- | ------------------------------------------------------------------------- |
| Admin Webapp   | POST   | `/notifications/send`              | Gửi thông báo đơn lẻ cho người dùng cụ thể (thường là phụ huynh/học sinh) |
| LMS Adapter    | POST   | `/notifications/schedule-reminder` | Gửi lời nhắc học tập định kỳ                                              |
| CRM Adapter    | POST   | `/notifications/bulk`              | Gửi thông báo hàng loạt (ví dụ: khai giảng, nghỉ học)                     |

---

## 📤 Các API được Notification Service phơi ra (Provider)

| Method | Endpoint                       | Mô tả                                                                                          |
| ------ | ------------------------------ | ---------------------------------------------------------------------------------------------- |
| GET    | `/notifications/{dispatch_id}` | Truy vấn trạng thái gửi của 1 dispatch cụ thể (cho phép trace từ service gọi như Admin Webapp) |

---

## 📥 Input Schema tiêu chuẩn cho gửi thông báo

```json
{
  "recipient_ids": ["u123", "u456"],
  "channels": ["email", "zalo", "app"],
  "template_code": "REMINDER_SCORE_AVAILABLE",
  "payload": {
    "student_name": "Nguyễn Văn A",
    "term": "HK1",
    "score_link": "https://..."
  },
  "priority": "high",
  "metadata": {
    "trace_id": "abc-xyz",
    "requested_by": "admin_user_id_123"  // ID của người gọi (user_id nếu là Admin Webapp, hoặc service_id nếu gọi nội bộ)
  }
}
```

> `template_code` xác định nội dung cụ thể (có thể đa ngôn ngữ) và hành vi từng kênh. `payload` là nội dung động được inject theo template.
> Trường `priority` (`normal` | `high`) có thể ảnh hưởng đến kênh gửi trước hoặc sau.
> `metadata.requested_by` dùng cho mục đích audit và trace, có thể là ID người dùng hoặc service.

---

## 📤 Output Schema (Response từ Notification Service)

```json
{
  "data": {
    "dispatch_id": "notif-xyz",
    "status": "queued",
    "channels": ["email", "zalo"],
    "failures": []
  },
  "error": null,
  "meta": {
    "trace_id": "abc-xyz",
    "timestamp": "...",
    "source": "notification_service"
  }
}
```

---

## 🛡️ RBAC & Phân quyền

* Chỉ những role có `SEND_NOTIFICATION_*` mới có thể gọi service này
* Gateway sẽ evaluate permission và forward header:

  * `X-User-ID`, `X-Role`, `X-Permissions`, `Trace-ID`
* Backend kiểm tra sự hiện diện của `SEND_NOTIFICATION_ALL` hoặc `SEND_NOTIFICATION_STUDENT`

---

## ⚠️ Lỗi đặc thù

| Code                  | Mô tả                                              |
| --------------------- | -------------------------------------------------- |
| `TEMPLATE_NOT_FOUND`  | `template_code` không tồn tại hoặc không được phép |
| `INVALID_RECIPIENT`   | Có `user_id` không hợp lệ hoặc bị block            |
| `CHANNEL_UNSUPPORTED` | Channel không khả dụng cho user                    |
| `RATE_LIMIT_EXCEEDED` | Vượt quá tần suất gửi cho user hoặc hệ thống       |

---

## ✅ Ghi chú

* `trace_id` phải được forward đầy đủ từ đầu chuỗi call để observability
* Mỗi `dispatch_id` có thể được truy vấn trạng thái qua `/notifications/{dispatch_id}` – API public cho trace và kiểm tra trạng thái gửi
* Template có thể versioned, hoặc hỗ trợ A/B testing nội bộ (được quy định trong `template_code` nâng cao)

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-011: API Error Format](../ADR/adr-011-api-error-format.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)

---

> “Thông báo tốt không chỉ đến đúng người – mà còn đúng lúc, đúng cách, và có thể kiểm soát được.”
