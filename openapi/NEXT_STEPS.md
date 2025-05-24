Tuyệt vời – việc hiện thực hóa OpenAPI Schema cho toàn bộ các Service Provider là bước quan trọng giúp:

* Đảm bảo **tài liệu API chính xác và đồng nhất** (ADR-009)
* Hỗ trợ frontend/dev team tích hợp nhanh
* Tích hợp **Contract Testing** và CI/CD (ADR-010)
* Kiểm soát và duy trì **versioning, deprecation** (ADR-013)

---

## ✅ Danh sách Service Provider cần sinh OpenAPI Schema

| STT | Service                   | Tên file IC                          | Trạng thái API             | Ghi chú                                     |
| --- | ------------------------- | ------------------------------------ | -------------------------- | ------------------------------------------- |
| 1   | **Auth Service**          | `ic-08-auth-service.md`              | Ổn định                    | Xác thực OTP, Google OAuth, refresh, logout |
| 2   | **User Service**          | `ic-09-user-service.md`              | Ổn định                    | RBAC, quản lý người dùng, audit             |
| 3   | **CRM Adapter**           | `ic-05-crm.md`                       | Ổn định                    | Lead management, public form, CRM → LMS     |
| 4   | **SIS Adapter**           | `ic-06-sis.md`                       | Ổn định                    | Student management, học phí, điểm danh      |
| 5   | **LMS Adapter**           | `ic-07-lms.md`                       | Ổn định                    | Assignment, timetable, scores               |
| 6   | **Notification Service**  | `ic-04-notification.md`              | Ổn định                    | Gửi Web/Zalo/Email/Chat notification        |
| 7   | **API Gateway**           | `ic-01-api-gateway.md`               | Ổn định                    | đại diện toàn bộ entrypoint công khai       |
| 8   | **Customer Portal (PWA)** | `ic-03-customer-portal.md`           | Client (Không là provider) | Không cần sinh OpenAPI                      |
| 9   | **Admin Webapp**          | `ic-02-admin-webapp.md`              | Client (Không là provider) | Không cần sinh OpenAPI                      |

---

## 📦 Đề xuất tiến trình ưu tiên

1. **Auth Service**
2. **User Service**
3. **CRM Adapter**
4. **SIS Adapter**
5. **LMS Adapter**
6. **Notification Service**
7. **API Gateway**

---

## 🎯 Mỗi file OpenAPI cần đảm bảo:

* [x] `openapi: 3.0.3`
* [x] `info`: title, version, contact
* [x] `paths`: đầy đủ method + description
* [x] `components/schemas`: tái sử dụng được (`UserOut`, `TokenPair`, `OtpRequest`, `ScoreOut`, ...)
* [x] `components/securitySchemes`: `BearerAuth` (JWT)
* [x] `components/responses`: `ErrorResponse`, `StandardResponse`
* [x] `tags`: theo module hoặc domain (`auth`, `user`, `notification`, ...)
* [x] Áp dụng chuẩn `ADR-011`, `ADR-012`, `ADR-013`, `ADR-009`

---
