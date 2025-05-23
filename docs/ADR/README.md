# 📚 DX VAS Architectural Decision Records (ADR)

Tài liệu này tổng hợp toàn bộ các quyết định thiết kế kỹ thuật (ADR) chính thức trong hệ thống **dx_vas**. Mỗi ADR phản ánh một quyết định quan trọng được thống nhất giữa các team liên quan nhằm đảm bảo kiến trúc hệ thống bền vững, bảo mật và dễ mở rộng.

---

## 🧭 Mục lục ADR theo chủ đề

### 1. 🧪 CI/CD & Deployment
- [ADR-001: CI/CD Strategy](./adr-001-ci-cd.md)
- [ADR-015: Deployment Strategy](./adr-015-deployment-strategy.md)
- [ADR-014: Zero Downtime Deployment](./adr-014-zero-downtime.md)
- [ADR-018: Release Approval Policy](./adr-018-release-approval-policy.md)
- [ADR-017: Environment Deploy Boundary](./adr-017-env-deploy-boundary.md)
- [ADR-016: Auto Scaling](./adr-016-auto-scaling.md)

### 2. 📡 API Design & Governance
- [ADR-004: Security Strategy](./adr-004-security.md)
- [ADR-006: Auth Strategy](./adr-006-auth-strategy.md)
- [ADR-007: RBAC Strategy](./adr-007-rbac.md)
- [ADR-009: API Governance](./adr-009-api-governance.md)
- [ADR-010: Contract Testing](./adr-010-contract-testing.md)
- [ADR-011: API Error Format](./adr-011-api-error-format.md)
- [ADR-012: API Response Structure](./adr-012-response-structure.md)
- [ADR-013: API Path Naming Convention](./adr-013-path-naming-convention.md)

### 3. 👁️ Observability & Reliability
- [ADR-005: Observability Strategy](./adr-005-observability.md)
- [ADR-008: Audit Logging](./adr-008-audit-logging.md)
- [ADR-020: Cost Observability](./adr-020-cost-observability.md)
- [ADR-021: External Observability](./adr-021-external-observability.md)
- [ADR-022: SLA/SLO Monitoring](./adr-022-sla-slo-monitoring.md)

### 4. 🏗️ Infrastructure & IaC
- [ADR-002: Infrastructure-as-Code (Terraform)](./adr-002-iac.md)
- [ADR-003: Secrets Management](./adr-003-secrets.md)
- [ADR-019: Project Layout (multi-project)](./adr-019-project-layout.md) ⚠️ *draft*

### 5. 🧬 Data Management & Governance
- [ADR-023: Schema Migration Strategy](./adr-023-schema-migration-strategy.md)
- [ADR-024: Data Anonymization & Retention](./adr-024-data-anonymization-retention.md)

---

## ✅ Quy ước đặt tên & trạng thái
- File được lưu tại `adr-XXX-name.md`
- Trạng thái có thể là: `accepted`, `draft`, `deprecated`
- Mỗi ADR có phần **Bối cảnh**, **Quyết định**, **Chi tiết kỹ thuật**, **Lợi ích**, **Rủi ro**, **Liên kết liên quan**

---

> “Good architecture decisions are not just documented — we live by them.”
