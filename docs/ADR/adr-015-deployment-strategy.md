---
id: adr-015-deployment-strategy
title: ADR-015: Chiến lược Triển khai cho hệ thống dx_vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [deployment, strategy, cloud-run, ci-cd, dx_vas]
---

## 📌 Bối cảnh

Hệ sinh thái **dx_vas** bao gồm nhiều loại dịch vụ:
- Dịch vụ frontend (SSR App Portal, Admin)
- Dịch vụ backend (API Gateway, LMS/CRM Adapter, Notification)
- Dịch vụ batch/ngầm hoặc tích hợp bên ngoài

Mỗi loại dịch vụ có đặc thù khác nhau về:
- Độ critical (ảnh hưởng người dùng thực, backend nội bộ...)
- Khả năng rollback
- Tần suất release

Việc chuẩn hoá **các chiến lược triển khai phù hợp cho từng loại** là cần thiết để đảm bảo:
- Tránh downtime, sai lệch version
- Có rollback rõ ràng nếu xảy ra lỗi
- Tối ưu chi phí và thời gian triển khai

---

## 🧠 Quyết định

**Áp dụng tập hợp các chiến lược triển khai chuẩn (manual, auto, canary, blue/green, progressive, rolling) cho hệ thống dx_vas. CI/CD sẽ hỗ trợ lựa chọn động tùy vào service và môi trường.**

---

## 🛠 Chi tiết các chiến lược triển khai

### 1. Manual Deployment
- **Mô tả**: Deploy thủ công bằng tay (UI/CLI) có kiểm soát người thực hiện
- **Sử dụng cho**:
  - Production critical service có release lớn
  - Trường hợp khẩn cấp cần kiểm soát cao
- **Ưu điểm**: Kiểm soát kỹ, an toàn hơn
- **Nhược điểm**: Dễ quên bước, không lặp lại được, dễ lỗi người

### 2. Automated Deployment (to Staging)
- **Mô tả**: Deploy tự động khi CI pass trên nhánh `dev`
- **Sử dụng cho**:
  - Staging của mọi service
- **Ưu điểm**: Nhanh, liên tục, phát hiện lỗi sớm
- **Nhược điểm**: Không kiểm soát production release

### 3. Rolling Update (Cloud Run Default)
- **Mô tả**: Cập nhật revision mới → từng request chuyển dần sang revision mới
- **Sử dụng cho**:
  - Service ít critical, không giữ state
- **Ưu điểm**: Đơn giản, không cấu hình thêm
- **Nhược điểm**: Không có bước kiểm soát traffic như Canary
- **Khả năng rollback**:
  - Cloud Run luôn tạo revision mới → rollback = update traffic về revision cũ
  - Nhưng rolling không có các bước chia traffic tinh vi như Canary hay môi trường song song như Blue/Green

### 4. Canary Release
- **Mô tả**: Triển khai phiên bản mới tới 1%/10% traffic → giám sát → tăng dần
- **Sử dụng cho**:
  - API Gateway, Frontend SSR, LMS Adapter
- **Ưu điểm**: Phát hiện sớm lỗi, rollback nhanh
- **Nhược điểm**: Phức tạp hơn trong CI/CD, cần alert/metric
- **Chi tiết rollout**:
  - `10%` → `25%` → `50%` → `100%`
  - Giám sát `error_rate`, `latency`, `availability`
  - Rollback bằng `update-traffic` về revision trước
- **Liên kết kỹ thuật**: [`adr-014-zero-downtime.md`](./adr-014-zero-downtime.md)

### 5. Blue/Green Deployment
- **Mô tả**: Triển khai vào môi trường tách biệt (green), kiểm thử → chuyển traffic toàn bộ từ blue → green nếu OK
- **Sử dụng cho**:
  - Thay đổi lớn, có thể gây mất tương thích
  - Khi cần kiểm thử thực tế nhưng không ảnh hưởng người dùng
- **Ưu điểm**: Rollback nhanh bằng cách switch lại blue
- **Nhược điểm**: Tốn tài nguyên (2 môi trường), khó duy trì state đồng bộ
- **Triển khai trên Cloud Run**:
  - Dùng `tag`: `blue`, `green`
  - Chuyển traffic bằng `gcloud run services update-traffic --to-tags`

### 6. Progressive Rollout (Gate-driven)
- **Mô tả**: Tăng dần traffic + trigger bởi review hoặc metric
- **Sử dụng cho**:
  - Các service có SLA cao, không thể chịu downtime
- **Ưu điểm**: Kiểm soát cực tốt, rollback từng bước
- **Nhược điểm**: Rất phức tạp, chỉ áp dụng khi cần nghiêm ngặt
- **Ví dụ**:
  - `5%` → review ✅
  - `25%` → monitor ✅
  - `100%`

---

## 🧭 Hướng dẫn lựa chọn chiến lược

| Tiêu chí | Manual | Auto | Rolling | Canary | Blue/Green | Progressive |
|---------|--------|------|---------|--------|------------|-------------|
| Critical service | ✅ | ❌ | ❌ | ✅ | ✅ | ✅✅ |
| Backend không giữ state | ✅ | ✅ | ✅ | ✅ | ✅ | ❌ |
| Có schema migration | ✅ | ❌ | ❌ | ✅ | ✅ | ✅ |
| Yêu cầu rollback nhanh | ❌ | ❌ | ✅ | ✅ | ✅ | ✅ |
| Nhỏ, internal, low risk | ❌ | ✅ | ✅ | ✅ | ❌ | ❌ |

---

## 🔁 Chiến lược rollback

| Chiến lược | Cách rollback |
|-----------|----------------|
| Manual | Revert thủ công từ Git/Cloud Run UI |
| Auto | Revert commit + re-trigger CI/CD |
| Rolling | Update traffic về revision trước (pin lại revision cũ) |
| Canary | Update traffic về revision trước |
| Blue/Green | Switch lại tag `blue` |
| Progressive | Ngừng rollout + rollback traffic |

> **Mọi rollback production phải có phê duyệt theo [`adr-018-release-approval-policy.md`]**

---

## 🔄 Tích hợp CI/CD

- GitHub Actions pipeline:
  - `dev` branch → auto deploy staging
  - `main` branch → deploy production theo chiến lược:
    - Có file `.deployment-strategy.yml`
    - CI quyết định: rolling / canary / blue/green
  - Canary monitor metric từ `Cloud Monitoring`
- Terraform modules hỗ trợ traffic split/tag Cloud Run revision

---

## 📎 Tài liệu liên quan

- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)
- Env Config: [ADR-005](./adr-005-env-config.md)
- Zero downtime: [ADR-014](./adr-014-zero-downtime.md)
- Release Approval Policy: [ADR-018](./adr-018-release-approval-policy.md)

---
> "Triển khai đúng không chỉ là push code – mà là kiểm soát rủi ro và tạo niềm tin."
