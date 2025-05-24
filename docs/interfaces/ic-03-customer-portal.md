# 📘 Interface Contract – Customer Portal (dx\_vas)

## 🧭 Mục tiêu

Tài liệu này định nghĩa hợp đồng giao tiếp cho **Customer Portal**, hệ thống dành cho phụ huynh và học sinh:

* Giao tiếp với API Gateway và backend LMS
* Áp dụng xác thực OTP hoặc Google Workspace
* Đảm bảo hành vi thống nhất về phân quyền, phản hồi lỗi, trace ID

---

## 🧩 Danh sách endpoint tiêu biểu (phụ huynh & học sinh)

| Method | Endpoint                 | Mô tả                                      | Input Schema / Params        | Output Schema        | Permission Code                                                                       |
| ------ | ------------------------ | ------------------------------------------ | ---------------------------- | -------------------- | ------------------------------------------------------------------------------------- |
| GET    | `/students/me`           | Xem thông tin học sinh (bản thân hoặc con) | —                            | `StudentOut`         | `VIEW_STUDENT_PROFILE_SELF` (học sinh) / `VIEW_STUDENT_PROFILE_OWN_CHILD` (phụ huynh) |
| GET    | `/students/me/scores`    | Xem điểm số học sinh                       | `term: str`, `subject?: str` | `List[ScoreOut]`     | `VIEW_STUDENT_SCORE_SELF` / `VIEW_STUDENT_SCORE_OWN_CHILD`                            |
| GET    | `/students/me/timetable` | Lịch học                                   | `week: ISODate`              | `List[TimetableOut]` | `VIEW_STUDENT_TIMETABLE_SELF` / `VIEW_STUDENT_TIMETABLE_OWN_CHILD`                    |
| GET    | `/notifications`         | Thông báo dành cho người dùng hiện tại     | `page: int`                  | `List[Notification]` | `VIEW_NOTIFICATION_SELF` / `VIEW_NOTIFICATION_OWN_CHILD`                              |

> Các endpoint `/students/me/...` được thiết kế để dùng chung cho cả học sinh và phụ huynh. Gateway xác định context người dùng và forward các permission code phù hợp: nếu là học sinh thì là `*_SELF`, nếu là phụ huynh thì là `*_OWN_CHILD`. Backend xử lý theo permission và ngữ cảnh `user_id` được forward.

---

## 🔐 Cơ chế xác thực & phân quyền

* Học sinh đăng nhập bằng Google Workspace → OAuth2
* Phụ huynh đăng nhập bằng OTP (email/SMS)
* Nhận JWT từ Auth Service (qua Gateway)
* Gửi `Authorization: Bearer <token>` trong mọi request
* Gateway sẽ forward các header:

  * `X-User-ID`, `X-Role`, `X-Auth-Method`, `X-Permissions`, `Trace-ID`

> Backend (LMS Adapter) sử dụng các permission code đã evaluate bởi Gateway, không tự re-evaluate điều kiện.

---

## 📦 Cấu trúc phản hồi

Tuân thủ theo [ADR-012](../ADR/adr-012-response-structure.md):

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

Khi lỗi:

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

## ✅ Ghi chú

* Customer Portal không cần hiển thị `trace_id`, nhưng có thể log lại nếu hỗ trợ debug / report
* UI nên hiển thị lỗi rõ ràng theo `error.message`, và hỗ trợ `error.code` để phân biệt các lỗi đặc thù như:

  * `OTP_INVALID`, `OTP_EXPIRED`, `STUDENT_NOT_FOUND`, `FORBIDDEN`
* Permission code `*_SELF` áp dụng cho học sinh; `*_OWN_CHILD` áp dụng cho phụ huynh. Tất cả được evaluate tại Gateway theo context `user_id`, không cần logic phân quyền ở backend (theo [ADR-007](../ADR/adr-007-rbac.md))

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-011: API Error Format](../ADR/adr-011-api-error-format.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [README.md – Phân quyền động cho phụ huynh](../README.md)

---

> “Customer Portal không chỉ là nơi tra cứu – mà là cầu nối giữa phụ huynh và hành trình học tập của học sinh.”
