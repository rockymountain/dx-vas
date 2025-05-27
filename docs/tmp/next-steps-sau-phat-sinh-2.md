## ✅ KẾT QUẢ HIỆN TẠI

Chúng ta đã:

* Cập nhật toàn bộ `README.md` theo kiến trúc mới
* Viết lại `rbac-deep-dive.md` theo mô hình Master – Sub Services
* Tái cấu trúc và sinh mới `system-diagrams.md` với 8 sơ đồ chuẩn hóa
* Làm rõ các tầng: Auth, RBAC, Notification, Tenant Stack, Pub/Sub, Deployment...

---

## 🔍 ĐỀ XUẤT RÀ SOÁT CÁC FILE ADR LIÊN QUAN

Tôi đã rà soát toàn bộ **24 file ADR** đã nộp từ `adr-001` đến `adr-024`.
Dưới đây là danh sách **các ADR nên được sửa hoặc bổ sung nhẹ** để đồng bộ:

---

### ✅ **CẦN CẬP NHẬT (bắt buộc)**

| ADR                                  | Tiêu đề             | Cần sửa gì?                                                                                                              |
| ------------------------------------ | ------------------- | ------------------------------------------------------------------------------------------------------------------------ |
| `adr-006-auth-strategy.md`           | Auth Strategy       | Bổ sung rõ luồng **JWT đa tenant**, vai trò Auth Master vs Sub Auth                                                      |
| `adr-007-rbac.md`                    | Dynamic RBAC        | Phải rewrite phần RBAC từ single-tenant → multi-tenant (gộp sơ đồ Master + Sub, `condition`, Pub/Sub cache invalidation) |
| `adr-008-audit-logging.md`           | Audit Logging       | Ghi rõ log theo `tenant_id`, phản hồi từ Sub Notification Service                                                        |
| `adr-015-deployment-strategy.md`     | Deployment Strategy | Ghi rõ mô hình **1 core + N tenant stack**, đề cập naming convention GCP projects                                        |
| `adr-019-project-layout.md`          | GCP Project Layout  | Phải ghi rõ: `dx-vas-core`, `dx-vas-tenant-abc`, `dx-vas-monitoring`, `dx-vas-data`... như sơ đồ 5                       |
| `adr-018-release-approval-policy.md` | Release Approval    | Thêm quy tắc cho Sub Service release theo tenant, phê duyệt riêng biệt                                                   |

---

### ⚠️ **CÂN NHẮC MỞ RỘNG (tuỳ chọn)**

| ADR                             | Gợi ý                                                                                   |
| ------------------------------- | --------------------------------------------------------------------------------------- |
| `adr-004-security.md`           | Bổ sung section về tenant isolation, JWT trust boundary                                 |
| `adr-003-secrets.md`            | Cần ghi rõ: tenant có thể dùng Zalo OA, Gmail creds riêng – nên được vault/rotate riêng |
| `adr-010-contract-testing.md`   | Có thể mở rộng: testing Sub Service phải dùng tenant context                            |
| `adr-022-sla-slo-monitoring.md` | Có thể tách alert theo tenant stack, log theo `tenant_id`                               |
