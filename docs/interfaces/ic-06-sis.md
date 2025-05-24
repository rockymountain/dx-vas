# 📘 Interface Contract – SIS Adapter (dx\_vas)

## 🧭 Mục tiêu

Tài liệu này mô tả hợp đồng giao tiếp (interface contract) của **SIS Adapter** – hệ thống quản lý học sinh chính thức:

* Quản lý thông tin hồ sơ học sinh và lớp học
* Đồng bộ dữ liệu với LMS, CRM và Admin Webapp
* Phục vụ cho các nghiệp vụ như đăng ký lớp, điều chuyển, thống kê

---

## 🧩 Endpoint SIS phơi ra (Provider API)

### 🎓 Học sinh

| Method | Endpoint                  | Mô tả                                          | Input Schema / Query Params                                                 | Output Schema      | Permission Code       |
|--------|---------------------------|------------------------------------------------|----------------------------------------------------------------------------|--------------------|------------------------|
| GET    | `/students`               | Danh sách học sinh                             | `page: int`, `limit: int`, `class_id?: UUID`, `status?: str`, `grade?: int` | `List[StudentOut]` | `VIEW_STUDENT_ALL`     |
| GET    | `/students/{id}`          | Thông tin chi tiết học sinh                    | —                                                                          | `StudentOut`       | `VIEW_STUDENT_DETAIL`  |
| POST   | `/students`               | Tạo học sinh mới (nhập học)                    | `StudentCreate`                                                            | `StudentOut`       | `CREATE_STUDENT`       |
| PATCH  | `/students/{id}`          | Cập nhật hồ sơ học sinh                        | `StudentUpdate`                                                            | `StudentOut`       | `EDIT_STUDENT`         |
| POST   | `/students/{id}/transfer` | Điều chuyển học sinh                           | `TransferRequest`                                                          | `StudentOut`       | `TRANSFER_STUDENT`     |
| POST   | `/students/batch`         | Lấy danh sách thông tin học sinh theo nhiều ID | `student_ids: List[UUID]`                                                  | `List[StudentOut]` | `VIEW_STUDENT_ALL`     |

### 🏫 Lớp học

| Method | Endpoint          | Mô tả                           | Input Schema / Query Params    | Output Schema    | Permission Code       |
|--------|-------------------|----------------------------------|-------------------------------|------------------|------------------------|
| GET    | `/classes`        | Danh sách lớp học               | `page: int`, `limit: int`     | `List[ClassOut]` | `VIEW_CLASS_ALL`       |
| GET    | `/classes/{id}`   | Thông tin chi tiết lớp học      | `id: UUID`                    | `ClassOut`       | `VIEW_CLASS_DETAIL`    |
| POST   | `/classes`        | Tạo lớp học mới                 | `ClassCreate`                 | `ClassOut`       | `CREATE_CLASS`         |
| PATCH  | `/classes/{id}`   | Cập nhật thông tin lớp học      | `ClassUpdate`                 | `ClassOut`       | `EDIT_CLASS`           |

> Tất cả endpoint đều đi qua API Gateway, yêu cầu JWT hợp lệ và header phân quyền chuẩn (`X-User-ID`, `X-Role`, `X-Permissions`, `Trace-ID`).

---

## 🔁 SIS gọi các hệ thống khác (API Consumer)

| Target           | Method | Endpoint                    | Mô tả                                                               |
|------------------|--------|-----------------------------|---------------------------------------------------------------------|
| Notification     | POST   | `/notifications/send`       | Gửi thông báo nhập học thành công hoặc chuyển lớp                  |
| LMS Adapter      | PUT    | `/sync/student/{id}`        | Đồng bộ thông tin hồ sơ học sinh mới hoặc đã cập nhật              |

---

## 🔐 Xác thực & RBAC

* Mọi endpoint gọi qua API Gateway → cần JWT
* Forward header gồm: `X-User-ID`, `X-Role`, `X-Permissions`, `Trace-ID`
* Mọi endpoint đều yêu cầu permission cụ thể → kiểm tra `X-Permissions` tại backend

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
    "source": "sis_adapter"
  }
}
```

---

## ⚠️ Lỗi đặc thù

| Code                | Mô tả                                                            |
|---------------------|------------------------------------------------------------------|
| `STUDENT_NOT_FOUND` | Không tồn tại học sinh với ID tương ứng                          |
| `INVALID_CLASS_ID`  | Class không tồn tại hoặc không thuộc campus người dùng phụ trách |
| `TRANSFER_CONFLICT` | Điều chuyển không hợp lệ do trạng thái học sinh                  |

---

## ✅ Ghi chú

* Mọi hành động tạo/cập nhật học sinh hoặc lớp đều tạo audit log
* Mọi thao tác cập nhật hồ sơ học sinh phải đồng bộ sang LMS
* Các trường `status` có thể là `admitted`, `studying`, `transferred`, `left`
* Các endpoint trả danh sách nên luôn hỗ trợ `page`, `limit` để đảm bảo hiệu năng và giảm tải frontend

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-011: API Error Format](../ADR/adr-011-api-error-format.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)

---

> “SIS là hệ tim của hệ thống giáo dục – nơi dữ liệu học sinh và lớp học luôn được duy trì chính xác, đầy đủ, và đồng bộ.”
