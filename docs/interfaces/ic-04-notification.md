# 📘 Interface Contract – Notification Service (dx\_vas)

## 🧭 Mục tiêu

Tài liệu này định nghĩa interface contract cho **Notification Service**, thành phần chịu trách nhiệm:

* Gửi thông báo qua email, app, Zalo, Google Chat
* Nhận yêu cầu gửi từ các hệ thống khác qua API Gateway
* Tự động hoá gửi theo template, định tuyến kênh gửi phù hợp

---

## 📤 Endpoint Notification Service phơi ra (Provider API)

| Method | Endpoint                       | Mô tả                                                                                          | Input Schema                          | Output Schema           | Permission Code                       |
|--------|--------------------------------|------------------------------------------------------------------------------------------------|---------------------------------------|--------------------------|----------------------------------------|
| POST   | `/notifications/send`          | Gửi thông báo đơn lẻ tới user cụ thể (Admin Webapp, SIS Adapter gọi)                         | `NotificationRequest`                 | `NotificationResult`    | `SEND_NOTIFICATION_ALL` / `SEND_NOTIFICATION_STUDENT` |
| POST   | `/notifications/schedule-reminder` | Gửi nhắc học bài định kỳ (LMS Adapter gọi)                                                   | `ReminderScheduleRequest`             | `NotificationResult`    | `SEND_NOTIFICATION_STUDENT`           |
| POST   | `/notifications/bulk`          | Gửi thông báo hàng loạt (CRM Adapter gọi)                                                     | `BulkNotificationRequest`             | `NotificationResult`    | `SEND_NOTIFICATION_ALL`               |
| GET    | `/notifications/{dispatch_id}` | Truy vấn trạng thái gửi của 1 dispatch cụ thể                                                 | `dispatch_id: str`                    | `NotificationStatusOut` | `VIEW_NOTIFICATION_STATUS`            |
| GET    | `/notifications`               | Lấy danh sách thông báo của người dùng hiện tại (dùng trong Customer Portal)                 | `page: int`                           | `List[NotificationOut]` | `VIEW_NOTIFICATION_SELF` / `VIEW_NOTIFICATION_OWN_CHILD` |

> Tất cả endpoint đều đi qua API Gateway và cần JWT hợp lệ. Gateway sẽ forward các header: `X-User-ID`, `X-Role`, `X-Permissions`, `Trace-ID`. Backend chỉ cần kiểm tra permission code có nằm trong `X-Permissions`.

---

## 📥 Input Schema

### NotificationRequest (POST /notifications/send)
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
    "requested_by": "admin_user_id_123"
  }
}
```

### ReminderScheduleRequest (POST /notifications/schedule-reminder)
```json
{
  "student_id": "abc123",
  "term": "HK1",
  "reminder_type": "assignment_due",
  "channels": ["app"],
  "template_code": "REMINDER_ASSIGNMENT",
  "payload": { "assignment_count": 2 },
  "schedule_time": "2025-07-10T18:00:00Z"
}
```

### BulkNotificationRequest (POST /notifications/bulk)
```json
{
  "audience_group": "new_parents_2025",
  "template_code": "WELCOME",
  "channels": ["zalo", "email"],
  "payload": {
    "event_name": "Khai giảng",
    "event_date": "2025-09-01"
  },
  "priority": "normal",
  "metadata": {
    "trace_id": "...",
    "requested_by": "crm_system"
  }
}
```

---

## 📤 Output Schema

### NotificationResult
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

### NotificationStatusOut (GET /notifications/{dispatch_id})
```json
{
  "dispatch_id": "notif-xyz",
  "status": "sent",
  "channels": ["email"],
  "failures": ["zalo"]
}
```

### List[NotificationOut] (GET /notifications)
```json
[
  {
    "id": "n001",
    "channel": "email",
    "title": "Điểm học kỳ 1 đã có",
    "received_at": "2025-07-10T19:00:00Z",
    "read": false
  },
  ...
]
```

---

## 🛡️ RBAC & Phân quyền

* Các permission code yêu cầu:
  - `SEND_NOTIFICATION_ALL` – cho phép gửi mọi loại thông báo
  - `SEND_NOTIFICATION_STUDENT` – chỉ cho phép gửi đến học sinh/phụ huynh
  - `VIEW_NOTIFICATION_SELF`, `VIEW_NOTIFICATION_OWN_CHILD` – xem thông báo (qua portal)
  - `VIEW_NOTIFICATION_STATUS` – xem trạng thái gửi
* API Gateway evaluate permission và forward các header tương ứng

---

## ⚠️ Lỗi đặc thù

| Code                  | Mô tả                                              |
|-----------------------|----------------------------------------------------|
| `TEMPLATE_NOT_FOUND`  | `template_code` không tồn tại hoặc không được phép |
| `INVALID_RECIPIENT`   | Có `user_id` không hợp lệ hoặc bị block            |
| `CHANNEL_UNSUPPORTED` | Channel không khả dụng cho user                    |
| `RATE_LIMIT_EXCEEDED` | Vượt quá tần suất gửi cho user hoặc hệ thống       |

---

## ✅ Ghi chú

* `trace_id` phải được forward xuyên suốt để đảm bảo observability
* Mỗi `dispatch_id` có thể được truy vấn qua `/notifications/{dispatch_id}`
* Template có thể versioned hoặc hỗ trợ A/B testing nội bộ nếu cần
* Các endpoint phải tuân thủ [ADR-012] và [ADR-011]

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-011: API Error Format](../ADR/adr-011-api-error-format.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)

---

> “Thông báo tốt không chỉ đến đúng người – mà còn đúng lúc, đúng cách, và có thể kiểm soát được.”
