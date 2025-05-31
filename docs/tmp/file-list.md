## 📁 Tổng hợp các file cung cấp

### 🏗️ Nhóm 1: Kiến trúc tổng quát & hệ thống

| Tên file                     | Tóm tắt nội dung                                                                 |
|-----------------------------|-----------------------------------------------------------------------------------|
| `README.md`                 | Giới thiệu tổng quan hệ thống chuyển đổi số VAS                                  |
| `index.md`                  | Mục lục dẫn link các ADR và tài liệu kỹ thuật                                    |
| `system-diagrams.md`        | Sơ đồ kiến trúc hệ thống DX-VAS với các nhóm thành phần chính                    |
| `rbac-deep-dive.md`         | Phân tích chuyên sâu về cơ chế phân quyền RBAC và chiến lược triển khai chi tiết |

---

### 📄 Nhóm 2: ADR – Architectural Decision Records

| Tên file                              | Tóm tắt nội dung                                                               |
|--------------------------------------|---------------------------------------------------------------------------------|
| `adr-001-ci-cd.md`                   | Quy ước triển khai CI/CD cho toàn hệ thống                                    |
| `adr-002-iac.md`                     | Hạ tầng dưới dạng mã (Infrastructure as Code)                                  |
| `adr-003-secrets.md`                 | Chính sách quản lý secrets và cấu hình an toàn                                 |
| `adr-004-security.md`                | Các nguyên tắc và quyết định bảo mật                                           |
| `adr-005-env-config.md`             | Chiến lược cấu hình môi trường runtime và deploy                               |
| `adr-006-auth-strategy.md`          | Chiến lược xác thực người dùng                                                 |
| `adr-007-rbac.md`                   | Thiết kế phân quyền động (RBAC)                                               |
| `adr-008-audit-logging.md`          | Cơ chế log hành vi người dùng phục vụ audit                                    |
| `adr-009-api-governance.md`         | Nguyên tắc thiết kế và quản trị API                                            |
| `adr-010-contract-testing.md`       | Áp dụng contract testing trong giao tiếp giữa các service                      |
| `adr-011-api-error-format.md`       | Chuẩn hóa định dạng lỗi API (ErrorResponse)                                    |
| `adr-012-response-structure.md`     | Chuẩn hóa cấu trúc response: `meta`, `data`, `error`                           |
| `adr-013-path-naming-convention.md` | Quy tắc đặt tên path/route trong API                                           |
| `adr-014-zero-downtime.md`          | Kỹ thuật zero downtime deployment                                              |
| `adr-015-deployment-strategy.md`    | Chiến lược triển khai đa môi trường                                            |
| `adr-016-auto-scaling.md`           | Quy định auto-scaling cho service                                              |
| `adr-017-env-deploy-boundary.md`    | Ranh giới môi trường triển khai (master/sub)                                   |
| `adr-018-release-approval-policy.md`| Quy trình xét duyệt trước khi release production                               |
| `adr-019-project-layout.md`         | Cấu trúc chuẩn cho layout thư mục dự án                                        |
| `adr-020-cost-observability.md`     | Giám sát chi phí vận hành                                                      |
| `adr-021-external-observability.md` | Tích hợp logging/monitoring với các dịch vụ ngoài                              |
| `adr-022-sla-slo-monitoring.md`     | Theo dõi SLA/SLO và tự động alert                                              |
| `adr-023-schema-migration-strategy.md`| Chiến lược migrate schema an toàn và version hóa DB                         |
| `adr-024-data-anonymization-retention.md` | Chính sách ẩn danh và giữ dữ liệu                                           |
| `adr-025-multi-tenant-versioning.md`| Version hóa theo từng tenant                                                   |
| `adr-026-hard-delete-policy.md`     | Chính sách không sử dụng hard delete                                           |
| `adr-027-data-management-strategy.md`| Chiến lược tổng thể quản lý dữ liệu, metadata, định danh, phân vùng           |

---

### 🔧 Nhóm 3: `user-service/master/` – Thiết kế service

| Tên file                  | Tóm tắt nội dung                                                          |
|---------------------------|---------------------------------------------------------------------------|
| `design.md`              | Thiết kế tổng thể logic, luồng, vai trò các thành phần                    |
| `data-model.md`          | Mô hình dữ liệu chi tiết cho UserGlobal, Tenant, Assignment, RBAC         |
| `interface-contract.md`  | Đặc tả contract: API, permission, sự kiện, audit                          |
| `openapi.yaml`           | File OpenAPI mô tả toàn bộ API, schema, chuẩn hóa ADR                    |
| `openapi.standard.md`    | Chuẩn 5★ để kiểm tra OpenAPI tuân thủ toàn bộ ADR và quy tắc nội bộ       |

---
