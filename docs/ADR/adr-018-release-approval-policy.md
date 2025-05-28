---
id: adr-018-release-approval-policy
title: ADR-018 <br> Chính sách phê duyệt và rollback release cho môi trường production của dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [release, approval, rollback, dx-vas, production]
---

# ADR-018: Chính sách Phê duyệt Phát hành (Release Approval Policy)

## Bối cảnh

Hệ thống dx-vas vận hành theo mô hình multi-tenant. Mỗi tenant là một trường thành viên, có stack riêng (frontend, backend, adapter), được deploy trên GCP project riêng.

Các dịch vụ core (Gateway, Auth Master, User Master...) được chia sẻ giữa các tenant và cần độ ổn định cao. Vì vậy, hệ thống cần một chính sách phê duyệt phát hành rõ ràng, hỗ trợ:

- Phát hành nhanh cho các tenant (khi cần fix lỗi riêng)
- Phê duyệt chặt chẽ cho các thay đổi hệ thống dùng chung
- Tự động hoá pipeline CI/CD nhưng vẫn đảm bảo kiểm soát chất lượng

## Quyết định

### 1. Tách pipeline theo nhóm

| Nhóm dịch vụ | Ví dụ | Phê duyệt |
|--------------|-------|-----------|
| **Core Services** | Gateway, Auth Master, User Master, Notification Master | Phải phê duyệt thủ công trên môi trường staging trước khi lên production |
| **Tenant Services** | Sub Auth, Sub User, Frontend Tenant A/B | Có thể tự động nếu qua test, hoặc yêu cầu duyệt tùy tenant |

### 2. Nguyên tắc phê duyệt

- **Core services**:
  - Phải có staging riêng
  - Phải chạy toàn bộ test (unit + integration)
  - Chỉ release sau khi được Superadmin hoặc DevOps Lead duyệt thủ công

- **Tenant stack**:
  - Mỗi tenant có môi trường staging riêng (VD: `staging-tenant-a.dx-vas.vn`)
  - Có thể phê duyệt tự động nếu:
    - Thay đổi không ảnh hưởng schema / auth / rbac
    - CI/CD chạy pass toàn bộ kiểm thử
  - Nếu thay đổi lớn → yêu cầu duyệt bởi quản trị viên kỹ thuật tenant (hoặc Superadmin nếu nghiêm trọng)

### 3. Công cụ và audit

- Sử dụng GitHub Actions + Approval Gates + môi trường `environments`
- Mọi hành động phê duyệt đều được log trong GitHub và chuyển về Cloud Audit
- Có webhook gửi notification khi release thành công cho từng tenant (qua Zalo/Slack)

### 4. Triển khai đa môi trường

- `dev`: tự động phát hành sau khi merge → để test nhanh
- `staging`: môi trường trung gian, bắt buộc kiểm thử thủ công với dữ liệu thật (clone)
- `prod`: môi trường chính thức, chỉ phát hành sau phê duyệt (core và tenant)

## Hệ quả

✅ Ưu điểm:

- Kiểm soát chặt chẽ các thay đổi ảnh hưởng toàn hệ thống
- Tenant vẫn có sự linh hoạt cao để phát hành riêng
- Phân biệt rõ trách nhiệm quản lý giữa Superadmin và quản trị tenant
- Tối ưu CI/CD theo mô hình tách stack

⚠️ Lưu ý:

- Cần quy trình rollback nhanh nếu release lỗi
- Pipeline phức tạp hơn nếu số tenant tăng nhiều → nên gom pipeline theo template

## Liên kết liên quan

- [`adr-015-deployment-strategy.md`](./adr-015-deployment-strategy.md)
- [`adr-019-project-layout.md`](./adr-019-project-layout.md)
- [`README.md#11-ci-cd--release-approval`](../README.md#11-ci-cd--release-approval)

---
> "Một bản release tốt bắt đầu từ sự chuẩn bị kỹ và kết thúc bằng khả năng rollback tốt hơn."
