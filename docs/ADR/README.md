Dưới đây là danh sách **các ADR từ API Gateway có thể tái sử dụng toàn phần hoặc điều chỉnh nhẹ để dùng cho toàn bộ hệ thống `dx_vas`**, phân loại theo **phạm vi áp dụng**:

---

## ✅ **Danh sách ADR tái sử dụng được cho `dx_vas`**

| ADR Code  | Tiêu đề                                   | Phạm vi tái sử dụng                              | Ghi chú chỉnh sửa cần thiết                                               |
| --------- | ----------------------------------------- | ------------------------------------------------ | ------------------------------------------------------------------------- |
| `adr-003` | CI/CD structure with GitHub Actions       | ✅ Toàn hệ thống                                  | Chuyển `service = api-gateway` → biến tham số; dùng cho frontend, backend |
| `adr-004` | API Versioning Strategy (`/api/v1/`)      | ✅ Gateway + tất cả backend API                   | Giữ nguyên, chỉ mở rộng thêm cho frontend/mobile                          |
| `adr-005` | Observability (Logging, Metrics, Tracing) | ✅ Toàn hệ thống                                  | Tách metric & log theo service/module                                     |
| `adr-006` | Auth via Google OAuth2 + JWT              | 🔶 Một phần                                      | Chỉ áp dụng nếu các service dùng chung gateway auth                       |
| `adr-007` | Error Handling chuẩn hóa JSON             | ✅ Toàn bộ API (gateway + backend service)        | Có thể chuẩn hóa luôn cho LMS adapter, CRM service...                     |
| `adr-008` | Rate Limiting với Redis                   | 🔶 Backend API + Gateway                         | Áp dụng được nếu các service chạy public-facing                           |
| `adr-009` | Security Hardening                        | ✅ Tất cả service (API, frontend, background job) | Cần bổ sung thêm cho frontend nếu dùng Next.js/Nuxt                       |
| `adr-010` | Deployment Strategy (Blue/Green + Canary) | ✅ Cloud Run service bất kỳ                       | Áp dụng y hệt cho LMS Adapter, Notification Service...                    |
| `adr-011` | Secrets Rotation                          | ✅ Toàn hệ thống                                  | Áp dụng chung cho frontend, backend, CI/CD                                |
| `adr-013` | Autoscaling Strategy (Cloud Run)          | ✅ Mọi service Cloud Run                          | Cần tinh chỉnh concurrency theo từng workload                             |
| `adr-014` | Multi-Environment Config                  | ✅ Frontend, Backend, Terraform                   | Cấu hình `ENV`, `.env`, secret injection → giữ nguyên                     |
| `adr-015` | Cost Observability                        | ✅ Mọi resource GCP                               | Chỉ cần sửa label thành `application = dx-vas-*`                          |
| `adr-016` | Resilience & Fallback                     | ✅ Mọi service có call external                   | Áp dụng luôn cho LMS Adapter, CRM adapter                                 |
| `adr-017` | Caching Strategy (Memory + Redis)         | ✅ Gateway, CRM adapter, frontend SSR             | Cần chú thích rõ scope nào không dùng được Redis                          |
| `adr-018` | API Governance (OpenAPI, lint, version)   | ✅ Tất cả service public API                      | Có thể dùng linter cho frontend GraphQL/REST nếu applicable               |
| `adr-019` | Contract Testing (Pact)                   | 🔶 Gateway ↔ Frontend, LMS Adapter ↔ LMS         | Dễ tái dùng nhưng cần người tiêu dùng rõ ràng                             |
| `adr-020` | API Lifecycle & Deprecation               | ✅ Tất cả API public                              | Không cần thay đổi, chỉ thêm endpoint `/docs/deprecation` vào portal      |
| `adr-021` | Zero-Downtime Deployment                  | ✅ Mọi service dùng Cloud Run                     | Giữ nguyên, tách bạch với chiến lược canary cụ thể                        |
| `adr-022` | Observability cho bên thứ ba              | ✅ LMS, Zalo, CRM, Firebase...                    | Có thể tách riêng `partner_name` cho từng tích hợp                        |
| `adr-023` | IaC Terraform Strategy                    | ✅ Toàn bộ GCP infra                              | Giữ nguyên – chỉ cần mở rộng module cho frontend infra                    |

---

## 🔶 **ADR chỉ tái dùng một phần / cần mở rộng**

| ADR                           | Lý do giới hạn                                                            |
| ----------------------------- | ------------------------------------------------------------------------- |
| `adr-001` (FastAPI framework) | Chỉ áp dụng cho backend Python service; frontend có thể dùng Next.js      |
| `adr-002` (RBAC động)         | Áp dụng chủ yếu cho API Gateway; các service khác có thể gọi RBAC qua API |

---

## ✅ Gợi ý nhóm ADR khi refactor sang `dx_vas`

| Nhóm                 | Mã ADR đề xuất mới                                                                                                           |
| -------------------- | ---------------------------------------------------------------------------------------------------------------------------- |
| DevOps/Infra         | `dx-adr-001-ci-cd`, `dx-adr-002-iac`, `dx-adr-003-env-config`, `dx-adr-004-secrets`                                          |
| Security             | `dx-adr-010-security`, `dx-adr-011-auth-strategy`                                                                            |
| Observability        | `dx-adr-020-logging`, `dx-adr-021-cost`, `dx-adr-022-3rdparty-tracing`                                                       |
| API & Design         | `dx-adr-030-api-version`, `dx-adr-031-api-governance`, `dx-adr-032-contract-test`, `dx-adr-033-deprecation`                  |
| Resilience & Scaling | `dx-adr-040-resilience`, `dx-adr-041-autoscaling`, `dx-adr-042-zero-downtime`, `dx-adr-043-rate-limit`, `dx-adr-044-caching` |

