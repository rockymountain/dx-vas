# 🧭 ADR Index – Tổng hợp & Phân loại Kiến trúc Quyết định cho dx-vas

Tài liệu này giúp bạn định hướng nhanh toàn bộ các quyết định kiến trúc (Architecture Decision Records – ADR) đã ban hành trong hệ thống **dx-vas**, phục vụ cho việc tra cứu, onboarding, và thiết kế service interface.

### 🔰 Chú giải biểu tượng

* ✅: Ảnh hưởng trực tiếp đến Interface Contract – cần tuân thủ khi thiết kế API (schema, header, cấu trúc response).
* ⚠️: Có ảnh hưởng gián tiếp – cần lưu ý trong behavior, versioning, rollout, hoặc integration.
* ❌: Không ảnh hưởng đến Interface Contract.

---

## 📚 Danh sách ADR theo nhóm chủ đề

### CI/CD & DevOps

| ADR                                             | Tên                     | Ảnh hưởng Interface Contract | Ghi chú / Chi tiết ảnh hưởng                                                                                                                              |
| ----------------------------------------------- | ----------------------- | ---------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [ADR-001](../ADR/adr-001-ci-cd.md)                   | CI/CD Strategy          | ✅                            | Mapping nhánh Git → môi trường triển khai → `ENV`, ảnh hưởng tới cách load cấu hình & endpoint                                                            |
| [ADR-002](../ADR/adr-002-iac.md)                     | IaC Terraform           | ⚠️                           | Resource như DB/Redis do IaC tạo có thể ảnh hưởng đến khả năng tương thích API, đặc biệt khi liên quan tới enum, constraint, hoặc storage layout          |
| [ADR-003](../ADR/adr-003-secrets.md)                 | Secrets Management      | ⚠️                           | Client cần biết cách lấy và sử dụng token (OAuth2 bearer token trong header); luồng OAuth phải public rõ `client_id`, `redirect_uri` nếu là public client |
| [ADR-005](../ADR/adr-005-env-config.md)              | Env Config              | ✅                            | `ENV=staging` → cấu hình phù hợp để gọi đúng backend, bật đúng flag                                                                                       |
| [ADR-017](../ADR/adr-017-env-deploy-boundary.md)     | Env Deploy Boundary     | ✅                            | Contract có thể khác biệt nhỏ giữa môi trường (một số API chỉ enable ở dev, flag bật/tắt theo ENV ảnh hưởng behavior)                                     |
| [ADR-018](../ADR/adr-018-release-approval-policy.md) | Release Approval Policy | ❌                            | Không ảnh hưởng trực tiếp contract                                                                                                                        |
| [ADR-015](../ADR/adr-015-deployment-strategy.md)     | Deployment Strategy     | ⚠️                           | Nếu rollout dần (canary, blue/green) → contract phải backward compatible trong suốt quá trình                                                             |
| [ADR-014](../ADR/adr-014-zero-downtime.md)           | Zero Downtime           | ✅                            | Không được breaking change trong API contract – phải tạo version mới nếu cần                                                                              |
| [ADR-016](../ADR/adr-016-auto-scaling.md)            | Auto Scaling            | ❌                            | Không ảnh hưởng contract                                                                                                                                  |
| [ADR-020](../ADR/adr-020-cost-observability.md)      | Cost Observability      | ❌                            | Không ảnh hưởng contract                                                                                                                                  |

### Auth, RBAC, API

| ADR                                            | Tên                    | Ảnh hưởng Interface Contract | Ghi chú / Chi tiết ảnh hưởng                                                                         |
| ---------------------------------------------- | ---------------------- | ---------------------------- | ---------------------------------------------------------------------------------------------------- |
| [ADR-006](../ADR/adr-006-auth-strategy.md)          | Auth Strategy          | ✅                            | Yêu cầu JWT/OTP; định nghĩa các header `X-User-ID`, `X-Role`, `X-Auth-Method`                        |
| [ADR-007](../ADR/adr-007-rbac.md)                   | RBAC Dynamic           | ✅                            | `X-Permissions`: danh sách các permission code đã evaluate; backend sử dụng và tin cậy danh sách này |
| [ADR-004](../ADR/adr-004-security.md)               | Security Strategy      | ✅                            | Header bảo mật (HTTPS, X-Frame-Options), sanitize input, CSRF nếu dùng form, rate-limit headers      |
| [ADR-009](../ADR/adr-009-api-governance.md)         | API Governance         | ✅                            | Bắt buộc versioning, OpenAPI, backward compatibility khi thay đổi                                    |
| [ADR-013](../ADR/adr-013-path-naming-convention.md) | Path Naming Convention | ✅                            | Tên path chuẩn RESTful, động từ, snake\_case, plural nouns                                           |
| [ADR-010](../ADR/adr-010-contract-testing.md)       | Contract Testing       | ✅                            | Interface contract phải có test tương ứng (Pact hoặc OpenAPI validation)                             |

### Logging, Error, Observability

| ADR                                            | Tên                    | Ảnh hưởng Interface Contract | Ghi chú / Chi tiết ảnh hưởng                                                                  |
| ---------------------------------------------- | ---------------------- | ---------------------------- | --------------------------------------------------------------------------------------------- |
| [ADR-011](../ADR/adr-011-api-error-format.md)       | API Error Format       | ✅                            | Mọi lỗi phải trả theo schema `error.code`, `message`, `details`                               |
| [ADR-012](../ADR/adr-012-response-structure.md)     | Response Structure     | ✅                            | Response luôn có `data`, `error`, `meta` – phải đúng định dạng này                            |
| [ADR-008](../ADR/adr-008-audit-logging.md)          | Audit Logging          | ✅                            | Hành động thay đổi dữ liệu phải sinh audit log, cần metadata chuẩn trong request              |
| [ADR-021](../ADR/adr-021-external-observability.md) | External Observability | ⚠️                           | Trace ID, correlation ID phải được truyền đầy đủ để log/trace đúng trong hệ thống ngoài       |
| [ADR-022](../ADR/adr-022-sla-slo-monitoring.md)     | SLA/SLO Monitoring     | ⚠️                           | Yêu cầu giới hạn latency/availability có thể ảnh hưởng API schema, streaming, hoặc pagination |

### Data, Storage, Governance

| ADR                                                  | Tên                            | Ảnh hưởng Interface Contract | Ghi chú / Chi tiết ảnh hưởng                                                                      |
| ---------------------------------------------------- | ------------------------------ | ---------------------------- | ------------------------------------------------------------------------------------------------- |
| [ADR-023](../ADR/adr-023-schema-migration-strategy.md)    | Schema Migration               | ✅                            | Contract phải backward compatible trong quá trình thay đổi DB schema (VD: thêm enum mới, cột mới) |
| [ADR-024](../ADR/adr-024-data-anonymization-retention.md) | Data Anonymization & Retention | ⚠️                           | Không trả PII không cần thiết; cần masking trong log/response nếu là môi trường dev/sandbox       |

### Tổ chức hệ thống

| ADR                                    | Tên                | Ảnh hưởng Interface Contract | Ghi chú / Chi tiết ảnh hưởng              |
| -------------------------------------- | ------------------ | ---------------------------- | ----------------------------------------- |
| [ADR-019](../ADR/adr-019-project-layout.md) | GCP Project Layout | ❌                            | Tổ chức hạ tầng, không ảnh hưởng contract |

---

## 🔍 Cách sử dụng bảng này khi viết Interface Contract

* Kiểm tra endpoint có tuân thủ `ADR-012` (format response) và `ADR-011` (format lỗi)?
* Request có cần `X-Permissions`, `X-User-ID`, `Trace-ID` theo `ADR-006`, `ADR-007`, `ADR-004` không?
* Path có theo `ADR-013`? Permission tương ứng có định nghĩa không (ADR-007)?
* Có ảnh hưởng bởi `contract testing` bắt buộc (ADR-010)?

---

> “Interface tốt không chỉ là API đúng – mà là API phản ánh chuẩn kiến trúc hệ thống.”
