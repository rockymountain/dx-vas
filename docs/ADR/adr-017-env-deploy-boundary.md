---
id: adr-017-env-deploy-boundary
title: ADR-017: Quy tắc triển khai theo môi trường cho hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [environment, deployment, boundary, dx-vas, config]
---

## 📌 Bối cảnh

Hệ thống **dx-vas** vận hành qua nhiều môi trường:
- `dev`: phát triển, test cá nhân, staging nội bộ
- `staging`: kiểm thử tích hợp, pre-prod, trình diễn
- `production`: môi trường chính thức phục vụ người dùng thực
- `sandbox`: dùng cho API công khai bên ngoài

Các lỗi triển khai thường đến từ:
- Deploy sai config môi trường
- Rollback nhầm giữa staging và production
- Không có quy tắc rõ ràng về quy trình release / điều kiện cho mỗi môi trường

---

## 🧠 Quyết định

**Xác định rõ ràng boundary và điều kiện triển khai cho từng môi trường trong hệ thống dx-vas, bao gồm quyền hạn, luồng CI/CD, cấu hình, và các yêu cầu kiểm thử / phê duyệt.**

---

## 🧩 Quy tắc triển khai theo môi trường

| Môi trường | Mục đích | Triển khai từ branch | Điều kiện deploy | Phê duyệt cần thiết |
|------------|----------|----------------------|-------------------|----------------------|
| `dev` | Phát triển & test | `feature/*`, `dev` | CI pass | Không |
| `staging` | Kiểm thử tích hợp | `dev`, `release/*` | CI pass + contract test pass | Không |
| `production` | Public release chính thức | `main` | CI pass + manual approval + monitoring ready | ✅ Theo [`adr-018-release-approval-policy.md`](./adr-018-release-approval-policy.md) |
| `sandbox` | Thử nghiệm API public | `sandbox` hoặc tách riêng từ `main` | CI pass | ✅ Đội hạ tầng approve |

---

## 🔒 Quy tắc bảo vệ môi trường production

- Chỉ nhóm `Platform` hoặc người có quyền release mới được merge vào `main`
- Pipeline deploy production yêu cầu:
  - Check list đầy đủ:
    - ✅ Migration đã chạy tách biệt
    - ✅ Contract test pass
    - ✅ Canary OK (nếu áp dụng)
    - ✅ Alert đã thiết lập và theo dõi
    - ✅ Tài liệu release note
  - Phê duyệt người có vai trò `release approver`
- Sau deploy phải có:
  - Tag version (`v1.4.0`, ...) trong Git
  - Rollback script khả dụng (nếu cần), theo hướng dẫn rollback trong [`adr-015-deployment-strategy.md`](./adr-015-deployment-strategy.md)

---

## 🔧 Cấu hình runtime theo môi trường

- Dùng file `.env`, `config.json` hoặc biến môi trường
- Cấu hình sẽ khác theo env:
  - DB endpoint
  - JWT signing key
  - Email, webhook, storage bucket
  - External API key (Zalo, Firebase...)
- File ví dụ:
```bash
.env.development
.env.staging
.env.production
```
- Được load bằng `pydantic-settings`, `dotenv`, hoặc config server
- Xem thêm hướng dẫn chi tiết trong [`adr-005-env-config.md`](./adr-005-env-config.md)

---

## ✅ Lợi ích

- Hạn chế lỗi do triển khai nhầm môi trường
- Rõ ràng quyền và trách nhiệm theo từng env
- Dễ thiết lập CI/CD pipeline riêng cho từng giai đoạn
- Chuẩn bị tốt hơn cho audit, rollback, tracking release

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Nhầm môi trường khi deploy | Pipeline enforce branch-env mapping |
| Rollback không chuẩn giữa env | Có file rollback riêng + tag rõ ràng version, tham khảo [`adr-015`](./adr-015-deployment-strategy.md) |
| Không tách biệt dữ liệu nhạy cảm giữa env | Dùng secrets riêng, Cloud Secret Manager theo env, tham khảo [`adr-003-secrets.md`](./adr-003-secrets.md) |

---

## 📎 Tài liệu liên quan

- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)
- Env Config: [ADR-005](./adr-005-env-config.md)
- Secrets: [ADR-003](./adr-003-secrets.md)
- Deployment Strategy: [ADR-015](./adr-015-deployment-strategy.md)
- Release Approval Policy: [ADR-018](./adr-018-release-approval-policy.md)

---
> “Một môi trường được bảo vệ tốt chính là lớp phòng thủ đầu tiên cho hệ thống production.”
