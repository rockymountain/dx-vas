# 📘 Interface Contract – CRM Adapter (dx-vas)

## 🧭 Mục tiêu

Tài liệu này định nghĩa giao tiếp của **CRM Adapter**, đóng vai trò:

* Nhận dữ liệu đồng bộ từ hệ thống tuyển sinh, webform landing page
* Cung cấp thông tin khách hàng (phụ huynh/học sinh tiềm năng) cho Admin Webapp
* Gửi thông tin cập nhật đến Notification Service nếu cần

---

## 🧩 Endpoint CRM phơi ra (Provider API)

| Method | Endpoint          | Mô tả                                | Input Schema / Query Params                                                                   | Output Schema   | Permission Code                       |
| ------ | ----------------- | ------------------------------------ | --------------------------------------------------------------------------------------------- | --------------- | ------------------------------------- |
| GET    | `/crm/leads`      | Danh sách lead (phụ huynh tiềm năng) | `source: str`, `status: str`, `assigned_to?: str`, `created_from?: date`, `created_to?: date` | `List[LeadOut]` | `VIEW_LEAD_ALL`                       |
| GET    | `/crm/leads/{id}` | Chi tiết một lead                    | `id: UUID`                                                                                    | `LeadOut`       | `VIEW_LEAD_DETAIL`                    |
| POST   | `/crm/leads`      | Tạo mới lead từ webform              | `LeadCreate`                                                                                  | `LeadOut`       | Không yêu cầu (public nếu là webform) |
| PATCH  | `/crm/leads/{id}` | Cập nhật trạng thái lead             | `LeadStatusUpdate`                                                                            | `LeadOut`       | `EDIT_LEAD_STATUS`                    |

> `LeadQueryParams` hỗ trợ lọc theo nguồn (`source`), trạng thái (`status`), người phụ trách (`assigned_to`), và khoảng thời gian tạo (`created_from`, `created_to`).

---

## 🔁 CRM gọi các hệ thống khác

| Target       | Method | Endpoint                   | Mô tả                                       |
| ------------ | ------ | -------------------------- | ------------------------------------------- |
| Notification | POST   | `/notifications/bulk`      | Gửi thông báo hàng loạt về chương trình mới |
| LMS Adapter  | PUT    | `/students/from-lead/{id}` | Khi lead chuyển thành học sinh chính thức   |

---

## 🔐 Xác thực & RBAC

* Tất cả request qua API Gateway → có JWT và header chuẩn:

  * `X-User-ID`, `X-Role`, `X-Permissions`, `Trace-ID`
* API `/crm/leads` (POST) là endpoint **public** dùng cho webform tuyển sinh:

  * Không yêu cầu JWT, nhưng **bắt buộc sử dụng token xác thực đơn giản (`X-Form-Token`) hoặc xác thực CAPTCHA v3** để ngăn abuse
  * Có thể áp dụng rate-limit per IP / session theo [ADR-004: Security Strategy](../ADR/adr-004-security.md)
  * Nếu abuse bị phát hiện → khóa nguồn hoặc trả về lỗi `RATE_LIMIT_EXCEEDED`
* Các endpoint còn lại yêu cầu xác thực và permission cụ thể (theo ADR-007)

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
    "source": "crm_adapter"
  }
}
```

---

## ⚠️ Lỗi đặc thù

| Code                     | Mô tả                                    |
| ------------------------ | ---------------------------------------- |
| `LEAD_NOT_FOUND`         | Không tồn tại lead tương ứng             |
| `INVALID_LEAD_SOURCE`    | Dữ liệu tạo lead thiếu hoặc không hợp lệ |
| `UNAUTHORIZED_EDIT_LEAD` | Không đủ quyền cập nhật lead             |
| `RATE_LIMIT_EXCEEDED`    | Webform bị spam vượt giới hạn hệ thống   |

---

## ✅ Ghi chú

* CRM Adapter là bridge giữa lead và hệ thống học sinh chính thức → dữ liệu phải rõ và được validate chặt
* Admin Webapp sẽ gọi các API này để xử lý luồng tuyển sinh
* Các thay đổi trạng thái quan trọng (ví dụ chuyển từ 'tiềm năng' → 'trúng tuyển') nên tạo `audit_log`

---

## 📎 Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-011: API Error Format](../ADR/adr-011-api-error-format.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)
* [ADR-004: Security Strategy](../ADR/adr-004-security.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)

---

> “CRM không chỉ lưu danh sách – mà kết nối con người với hành trình giáo dục.”
