# 📘 Interface Contract – LMS Adapter (dx\_vas)

## 🧭 Mục tiêu

Tài liệu này mô tả hợp đồng giao tiếp (interface contract) của **LMS Adapter** – hệ thống quản lý học tập:

* Cho phép tra cứu lịch học, điểm số, bài tập cho học sinh/phụ huynh
* Nhận đồng bộ dữ liệu từ SIS và CRM
* Cung cấp thông tin học tập cho Admin Webapp

---

## 🧩 Endpoint LMS phơi ra (Provider API)

| Method | Endpoint                    | Mô tả                                       | Input Schema / Query Params                                      | Output Schema         | Permission Code                         |
| ------ | --------------------------- | ------------------------------------------- | ---------------------------------------------------------------- | --------------------- | --------------------------------------- |
| GET    | `/scores/student/{id}`      | Lấy điểm học sinh theo kỳ                   | `term: str`, `subject?: str`, `page?: int`, `limit?: int`        | `List[ScoreOut]`      | `VIEW_STUDENT_SCORE_SELF/OWN_CHILD`     |
| GET    | `/timetable/student/{id}`   | Lịch học theo tuần                          | `week: ISODate`, `page?: int`, `limit?: int`                     | `List[TimetableOut]`  | `VIEW_STUDENT_TIMETABLE_SELF/OWN_CHILD` |
| GET    | `/assignments/student/{id}` | Danh sách bài tập                           | `status?: str`, `due_before?: date`, `page?: int`, `limit?: int` | `List[AssignmentOut]` | `VIEW_STUDENT_ASSIGNMENTS`              |
| PUT    | `/sync/student/{id}`        | Nhận đồng bộ thông tin từ SIS               | `StudentSyncPayload`                                             | `StudentOut`          | `SYNC_FROM_SIS`                         |
| PUT    | `/students/from-lead/{id}`  | Nhận học sinh từ CRM khi chuyển đổi từ lead | `StudentFromLeadPayload`                                         | `StudentOut`          | `CREATE_STUDENT_FROM_LEAD`              |

> Các endpoint `/student/{id}` dùng cho cả học sinh và phụ huynh (Gateway determine permission code phù hợp theo vai trò). Tất cả endpoint trả danh sách đều hỗ trợ phân trang `page`, `limit`.

---

## 🔁 LMS gọi hệ thống khác

| Target       | Method | Endpoint                           | Mô tả                         |
| ------------ | ------ | ---------------------------------- | ----------------------------- |
| Notification | POST   | `/notifications/schedule-reminder` | Gửi nhắc học bài, hạn bài tập |

---

## 🔐 Xác thực & RBAC

* Tất cả gọi qua API Gateway → JWT và các header:

  * `X-User-ID`, `X-Role`, `X-Permissions`, `Trace-ID`
* Backend chỉ cần kiểm tra `X-Permissions` để xác định quyền truy cập
* Học sinh và phụ huynh dùng cùng endpoint nhưng khác permission code
* Đối với các endpoint như `/students/from-lead/{id}` (gọi bởi CRM Adapter):

  * Đây là luồng **service-to-service**, không đại diện cho người dùng cuối cụ thể.
  * Vẫn đi qua Gateway, có thể sử dụng `X-User-ID` đại diện cho service CRM với role `system`.
  * Permission `CREATE_STUDENT_FROM_LEAD` được cấp cho CRM thông qua cấu hình RBAC.
  * LMS Adapter kiểm tra `X-Permissions` như bình thường, không cần phân biệt user/service.
  * `X-User-ID`, `X-Role`, `X-Permissions`, `Trace-ID`
* Backend chỉ cần kiểm tra `X-Permissions` để xác định quyền truy cập
* Học sinh và phụ huynh dùng cùng endpoint nhưng khác permission code

---

## 📦 Response structure

Tuân thủ [ADR-012](../ADR/adr-012-response-structure.md):

```json
{
  "data": {...},
  "error": null,
  "meta": {
    "trace_id": "...",
    "timestamp": "...",
    "source": "lms_adapter"
  }
}
```

---

## ⚠️ Lỗi đặc thù

| Code                             | Mô tả                                                                                                                                                                                                              |
| -------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `SCORE_NOT_FOUND`                | Không có điểm cho học sinh/kỳ được yêu cầu                                                                                                                                                                         |
| `ASSIGNMENT_NOT_FOUND`           | Không có bài tập tương ứng                                                                                                                                                                                         |
| `UNAUTHORIZED_ACCESS_TO_STUDENT` | Dữ liệu học sinh được yêu cầu không khớp với phạm vi được phép của người dùng (dù permission code đã được cấp). Thường do dữ liệu không đồng nhất hoặc logic nghiệp vụ sâu hơn ngoài phạm vi đánh giá của Gateway. |

---

## ✅ Ghi chú

* Mọi thay đổi từ SIS hoặc CRM sẽ override dữ liệu học sinh nếu gọi `/sync/student/{id}` hoặc `/students/from-lead/{id}`
* Hành vi trả điểm/bài tập/lịch học phải đồng bộ với permission hiện hành (SELF hoặc OWN\_CHILD)
* Mọi truy cập được audit theo ADR-008

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-011: API Error Format](../ADR/adr-011-api-error-format.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)

---

> “LMS không chỉ ghi điểm – mà ghi lại cả hành trình học tập của học sinh.”
