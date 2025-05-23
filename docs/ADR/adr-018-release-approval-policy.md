---
id: adr-018-release-approval-policy
title: ADR-018: Chính sách phê duyệt và rollback release cho môi trường production của dx_vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [release, approval, rollback, dx_vas, production]
---

## 📌 Bối cảnh

Việc release dịch vụ mới vào môi trường production của **dx_vas** là hành động có rủi ro cao, đặc biệt với các hệ thống như API Gateway, LMS Adapter, CRM Proxy và frontend SSR. Trong một số trường hợp trước đây, việc thiếu kiểm tra chéo hoặc rollback kịp thời đã gây:
- Downtime cho người dùng thực
- Mất session do JWT/token thay đổi không backward-compatible
- Conflict dữ liệu do schema không đồng bộ

Để đảm bảo tính ổn định và độ tin cậy, cần có **cơ chế phê duyệt bắt buộc trước khi release production**, cũng như **chiến lược rollback rõ ràng**.

---

## 🧠 Quyết định

**Áp dụng chính sách phê duyệt bắt buộc và kế hoạch rollback chuẩn hoá cho mọi bản release lên production trong hệ thống dx_vas.**

---

## 🔐 Chính sách phê duyệt release

### ✅ Áp dụng cho:
- Mọi service trong môi trường `production`
- Bao gồm cả backend (API Gateway, Adapter) và frontend (SSR WebApp)

### ✅ Người được phép approve:
- Thành viên thuộc nhóm `Platform`
- Hoặc người được gán quyền `release approver`
- Phải **khác người với người trực tiếp merge hoặc thực hiện deploy** (require 4-eyes)

### ✅ Cơ chế phê duyệt:
- Thực hiện qua GitHub Pull Request:
  - Merge vào nhánh `main` sẽ trigger release production
  - Phải có tối thiểu 1 approve từ người thuộc nhóm `release approver`
  - Check CI/CD pass + checklist hoàn tất
- Bắt buộc checklist trước khi phê duyệt:
  - ✅ Migration đã chạy (hoặc tách riêng)
  - ✅ Đã test staging xong
  - ✅ Canary hoặc QA OK (nếu dùng)
  - ✅ Tài liệu release note rõ ràng
  - ✅ Có hướng dẫn rollback cụ thể kèm theo (link đến script hoặc mô tả)

---

## ♻️ Chính sách rollback

| Chiến lược triển khai | Cách rollback |
|-----------------------|----------------|
| Rolling | Re-route traffic về revision cũ (Cloud Run) |
| Canary | Update traffic về revision ổn định trước đó |
| Blue/Green | Switch lại tag/môi trường `blue` |
| Manual | Revert Git commit + redeploy bản trước |

> Mọi rollback cần ghi lại lý do và gửi báo cáo post-mortem nếu ảnh hưởng production.

### Công cụ hỗ trợ:
- `gcloud run services update-traffic`
- Tag version cũ: `git tag v1.2.3`
- Rollback script nằm trong `/scripts/rollback/`
- CI/CD hỗ trợ `rollback.yml` workflow (nếu được trigger)

---

## 🛡️ Audit & Logging
- Tất cả hành động release & rollback phải được:
  - Ghi log vào hệ thống audit (Cloud Logging hoặc GitHub Audit Log)
  - Gắn trace ID và thông tin người thực hiện
- Các thay đổi có ảnh hưởng dữ liệu **phải ghi chú rõ phạm vi** trong PR

---

## ✅ Lợi ích

- Ngăn release vô tình hoặc không kiểm soát
- Giảm rủi ro khi deploy vào production
- Tăng tính minh bạch, trách nhiệm và khả năng truy vết

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Approval chậm gây chặn luồng release | Có danh sách người duyệt dự phòng, phân công rõ theo từng module |
| Bỏ qua rollback hoặc rollback sai cách | Checklist bắt buộc, CI cảnh báo nếu không có hướng dẫn rollback |
| Người duyệt không hiểu thay đổi | Đính kèm link mô tả, PR summary, ảnh chụp màn hình hoặc demo |

---

## 📎 Tài liệu liên quan

- Deployment Strategy: [ADR-015](./adr-015-deployment-strategy.md)
- Environment Policy: [ADR-017](./adr-017-env-deploy-boundary.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)
- Zero Downtime: [ADR-014](./adr-014-zero-downtime.md)

---
> "Một bản release tốt bắt đầu từ sự chuẩn bị kỹ và kết thúc bằng khả năng rollback tốt hơn."
