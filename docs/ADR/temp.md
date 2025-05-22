Dưới đây là danh sách đầy đủ các ADR thuộc **nhóm Deployment Strategy** mà bạn nên có cho hệ thống `dx_vas`, kèm theo **refactor tương ứng từ API Gateway (nếu có)**:

---

## ✅ Nhóm Deployment Strategy – Đề xuất ADR

| Mã ADR (dx\_vas)                     | Tiêu đề                                                                  | Refactor từ                                       | Ghi chú                                                        |
| ------------------------------------ | ------------------------------------------------------------------------ | ------------------------------------------------- | -------------------------------------------------------------- |
| `adr-014-zero-downtime.md`           | Chiến lược Triển khai không gián đoạn (Zero-Downtime Deployment)         | `API_Gateway/adr-021-zero-downtime-deployment.md` | ✅ Quan trọng nhất trong nhóm này                               |
| `adr-015-deployment-strategy.md`     | Chiến lược tổng thể cho các loại triển khai (manual, canary, blue/green) | (Kết hợp từ nhiều ADR)                            | Có thể mở rộng từ `adr-010-deployment-strategy.md` nếu tách ra |
| `adr-016-auto-scaling.md`            | Tự động scale dịch vụ Cloud Run                                          | `API_Gateway/adr-013-autoscaling-strategy.md`     | Xác định concurrency, min/max instance                         |
| `adr-017-env-deploy-boundary.md`     | Quy tắc deploy theo môi trường (dev, staging, prod)                      | Mở rộng từ `adr-014-multi-env-config.md`          | Xác định điều kiện deploy từng env                             |
| `adr-018-release-approval-policy.md` | Chính sách phê duyệt release & rollback                                  | (Mới)                                             | Define role, approval logic trước khi release production       |

---

## 🔄 Những ADR đã có liên quan

* `adr-001-ci-cd.md`: CI/CD pipeline — **đã có**, cần liên kết với các ADR trên
* `adr-005-env-config.md`: Định nghĩa biến môi trường — đã có, liên quan đến deploy
* `adr-002-iac.md`: Hạ tầng dưới dạng mã (Terraform) — hỗ trợ mạnh việc deploy multi-env

---

## 📌 Gợi ý trình tự triển khai tiếp theo

1. Bắt đầu từ: `adr-014-zero-downtime.md` — refactor trực tiếp từ ADR-021 (API Gateway)
2. Tiếp theo: `adr-016-auto-scaling.md` — refactor từ ADR-013
3. Rồi tới: `adr-017-env-deploy-boundary.md` — bổ sung cho `adr-005` để enforce production rules
4. Cuối cùng: `adr-018-release-approval-policy.md` — giúp kiểm soát việc release qua PR/CI

---

Bạn muốn mình bắt đầu tạo file `dx_vas/adr-014-zero-downtime.md` ngay bây giờ không?
