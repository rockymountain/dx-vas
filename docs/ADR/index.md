# 📘 ADR Index – dx-vas

> Danh sách và phạm vi áp dụng của các **Architecture Decision Records (ADR)** cho hệ thống dx-vas. Bảng dưới đây liệt kê từng ADR, các service bị ảnh hưởng và vai trò/điểm nhấn triển khai đặc biệt nếu có.

| ADR | Áp dụng cho Service | Vai trò/Điểm nhấn triển khai |
|-----|----------------------|-------------------------------|
| [ADR-001: CI/CD](./adr-001-ci-cd.md) | Tất cả services | – |
| [ADR-002: IaC](./adr-002-iac.md) | Tất cả services | – |
| [ADR-003: Secrets](./adr-003-secrets.md) | Tất cả services | – |
| [ADR-004: Security](./adr-004-security.md) | Tất cả services | ✅ API Gateway: rate-limiting, WAF; ✅ Backend: sanitize input, secure dependencies |
| [ADR-005: Env Config](./adr-005-env-config.md) | Tất cả services | – |
| [ADR-006: Auth Strategy](./adr-006-auth-strategy.md) | Auth Service, API Gateway | ✅ Auth Service: OTP & OAuth flow; ✅ API Gateway: validate & inject header |
| [ADR-007: RBAC](./adr-007-rbac.md) | User Service, API Gateway | ✅ API Gateway: evaluate RBAC at runtime; ✅ User Service: quản lý dữ liệu RBAC |
| [ADR-008: Audit Logging](./adr-008-audit-logging.md) | Tất cả services | ✅ User Service: expose /audit/logs; ✅ Gateway & Backend: log hành vi có side-effect |
| [ADR-009: API Governance](./adr-009-api-governance.md) | Tất cả services phơi API | ✅ Mỗi service phải cung cấp OpenAPI schema, được kiểm tra qua CI |
| [ADR-010: Contract Testing](./adr-010-contract-testing.md) | Tất cả services phơi API | – |
| [ADR-011: API Error Format](./adr-011-api-error-format.md) | Tất cả services phơi API | – |
| [ADR-012: Response Structure](./adr-012-response-structure.md) | Tất cả services phơi API | – |
| [ADR-013: Path Naming Convention](./adr-013-path-naming-convention.md) | Tất cả services phơi API | – |
| [ADR-014: Zero Downtime Deployment](./adr-014-zero-downtime.md) | Tất cả services | ✅ Yêu cầu Backward Compatible schema, tách deploy & migration |
| [ADR-015: Deployment Strategy](./adr-015-deployment-strategy.md) | Tất cả services | – |
| [ADR-016: Auto Scaling](./adr-016-auto-scaling.md) | Tất cả services | – |
| [ADR-017: Env Deploy Boundary](./adr-017-env-deploy-boundary.md) | Tất cả services | – |
| [ADR-018: Release Approval Policy](./adr-018-release-approval-policy.md) | Tất cả services | – |
| [ADR-019: Project Layout](./adr-019-project-layout.md) | Tất cả services | – |
| [ADR-020: Cost Observability](./adr-020-cost-observability.md) | Tất cả services | – |
| [ADR-021: External Observability](./adr-021-external-observability.md) | Tất cả services | – |
| [ADR-022: SLA, SLO, Monitoring](./adr-022-sla-slo-monitoring.md) | Tất cả services | – |
| [ADR-023: Schema Migration Strategy](./adr-023-schema-migration-strategy.md) | Các service có quản lý DB schema (User, Auth, SIS, LMS, CRM...) | ✅ Tuân thủ chiến lược 3 bước (Prepare – Transition – Cleanup) và rollbackable |
| [ADR-024: Data Anonymization & Retention](./adr-024-data-anonymization-retention.md) | Tất cả services xử lý PII (User, Auth, SIS, LMS, CRM, Gateway...) | ✅ TTL policy, masking PII logs, anonymize sandbox |

---

> 🔎 Gợi ý sử dụng: Nếu bạn triển khai service mới, hãy rà soát bảng này để đảm bảo service tuân thủ đầy đủ các ADR tương ứng và ghi rõ mọi điểm triển khai đặc biệt (nếu có) trong Interface Contract.
