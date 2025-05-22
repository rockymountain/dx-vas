Rất hay! Hãy rà soát lại nhóm **API & Design** để đảm bảo bạn không bỏ sót ADR quan trọng nào. Dưới đây là tổng hợp các ADR thuộc nhóm này trong hệ thống `dx_vas`, cùng trạng thái hiện tại:

---

## ✅ Nhóm **API & Design** – các ADR đã hoàn thành

| ADR                           | Tên                                        | Trạng thái      | Refactor từ                     |
| ----------------------------- | ------------------------------------------ | --------------- | ------------------------------- |
| `adr-004-security.md`         | Chiến lược Bảo mật tổng thể                | ✅ Đã hoàn thành | `adr-009-security-hardening.md` |
| `adr-005-env-config.md`       | Cấu hình đa môi trường                     | ✅ Đã hoàn thành | `adr-014-multi-env-config.md`   |
| `adr-006-auth-strategy.md`    | Xác thực người dùng (OAuth2 + JWT)         | ✅ Đã hoàn thành | `adr-006-auth-design.md`        |
| `adr-007-rbac.md`             | Phân quyền động (RBAC Strategy)            | ✅ Đã hoàn thành | `adr-002-rbac-design.md`        |
| `adr-008-audit-logging.md`    | Audit Logging xuyên suốt hệ thống          | ✅ Đã hoàn thành | `adr-012-audit-logging.md`      |
| `adr-009-api-governance.md`   | API Governance (versioning, OpenAPI, lint) | ✅ Đã hoàn thành | `adr-018-api-governance.md`     |
| `adr-010-contract-testing.md` | Contract Testing toàn hệ thống (Pact)      | ✅ Đã hoàn thành | `adr-019-contract-testing.md`   |

---

## 📌 Gợi ý **ADR còn thiếu** trong nhóm API & Design (có thể xem xét bổ sung):

| Mã đề xuất                          | Tên đề xuất                                    | Gợi ý                                                                         |
| ----------------------------------- | ---------------------------------------------- | ----------------------------------------------------------------------------- |
| `adr-011-api-error-format.md`       | Chuẩn hoá định dạng lỗi API                    | Trích từ `adr-007-error-handling.md` (API Gateway) – hiện chưa refactor       |
| `adr-012-response-structure.md`     | Response structure chuẩn `{data, error, meta}` | Một phần có trong `adr-009`, nhưng có thể tách thành ADR riêng nếu dùng chung |
| `adr-013-path-naming-convention.md` | Quy ước đặt tên path, method, tham số          | Nếu muốn quy định RESTful URL, query param consistent cho toàn hệ thống       |

---

## 📎 Lưu ý:

* Các ADR bổ sung này thường được xem là **phụ trợ** cho `adr-009-api-governance.md`, nhưng nếu bạn muốn các đội tuân thủ nghiêm ngặt và có thể kiểm tra tự động (lint, CI), thì nên tách riêng để dễ kiểm soát, review, và nâng cấp sau này.
* Nếu ADR `adr-007-error-handling.md` (API Gateway) là đủ rõ ràng và áp dụng toàn hệ thống, bạn có thể chọn refactor nó thành `dx_vas/adr-011-api-error-format.md`.

---

Bạn muốn mình bắt đầu với bản nào tiếp theo trong các ADR còn thiếu này? `adr-011-api-error-format.md` là ứng viên hợp lý để tiếp nối chuỗi API Design.
