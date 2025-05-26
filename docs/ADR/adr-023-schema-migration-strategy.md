---
id: adr-023-schema-migration-strategy
title: ADR-023: Chiến lược thay đổi schema dữ liệu an toàn và có thể rollback cho hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [schema, database, migration, compatibility, dx-vas]
---

## 📌 Bối cảnh

Nhiều thành phần trong hệ thống **dx-vas** sử dụng cơ sở dữ liệu quan hệ hoặc NoSQL (PostgreSQL, Firestore, Redis). Khi triển khai tính năng mới hoặc refactor, thay đổi schema là điều không thể tránh khỏi. Tuy nhiên, nếu làm không cẩn thận, migration có thể gây:
- Mất dữ liệu hoặc mất tương thích ngược (breaking change)
- Downtime nếu migration quá nặng hoặc xảy ra khi hệ thống đang online
- Không rollback được nếu không có chiến lược tách biệt và kiểm soát tốt

---

## 🧠 Quyết định

**Áp dụng chiến lược migration an toàn theo hướng forward-compatible, rollbackable và không gây downtime cho hệ thống dx-vas. Migration phải được quản lý độc lập với code deploy.**

---

## 🧱 Chiến lược triển khai migration theo 3 bước

### 1. **Prepare – Add, không thay thế**
- Thêm trường mới (nullable, optional), KHÔNG đổi tên hoặc xóa trực tiếp
- Không thay đổi kiểu dữ liệu đột ngột
- Nếu cần chuyển đổi dữ liệu, thêm cột mới và chuyển dần sang (shadow field)

### 2. **Transition – Ứng dụng hỗ trợ cả schema cũ & mới**
- Code phải tương thích cả hai schema (đọc cả field cũ và mới)
- Tạo flag/toggle để kiểm soát hành vi migration (tốt hơn nếu có experiment rollout)
- Chạy migration script riêng biệt, không gắn vào code deploy

### 3. **Cleanup – Chỉ sau khi đã hoàn tất rollout và verify**
- Chỉ xóa/rename schema cũ khi:
  - Không còn service nào sử dụng
  - Monitoring xác nhận không có traffic dùng field cũ
  - Đã rollback được migration data nếu cần

---

## 🧰 Công cụ & Quy trình kỹ thuật

- Migration script nên sử dụng:
  - Alembic (PostgreSQL)
  - Firebase Admin SDK (Firestore)
  - Custom script Python/NodeJS (Redis, Mongo, etc.)
- Mỗi migration có:
  - ID duy nhất, version hóa theo timestamp
  - Chạy qua `pre-release checklist`
  - Có trạng thái: `pending`, `running`, `done`, `rolled_back`
- Script lưu log tại Cloud Logging + BigQuery để audit
- Rollback script đi kèm migration (nếu destructive)

---

## ⏱ Thời điểm chạy migration

| Loại migration | Thời điểm chạy | Ghi chú |
|----------------|----------------|--------|
| Add column/index | Trước deploy | Không ảnh hưởng code cũ |
| Populate data mới | Sau deploy | Code mới có thể đọc field đó |
| Drop/rename field | Sau nhiều ngày | Phải chắc chắn backward compat đã được loại bỏ |

---

## ✅ Lợi ích

- Migration không làm gián đoạn hệ thống
- Có khả năng rollback nếu deployment thất bại
- Cho phép rollout thay đổi lớn theo từng giai đoạn, không buộc all-or-nothing

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Chạy migration quá lâu gây lock | Batch + timeout hợp lý, chạy nền (async worker) |
| Migration lén gắn vào code deploy | CI/CD enforce separation, checklist bắt buộc |
| Không rollback được khi rename/drop | Dùng shadow field + delay cleanup + backup trước khi drop |

---

## 📎 Tài liệu liên quan

- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)
- Release Approval: [ADR-018](./adr-018-release-approval-policy.md)
- Deployment Strategy: [ADR-015](./adr-015-deployment-strategy.md)
- Schema versioning (nếu có): Sẽ bổ sung nếu dùng Proto hoặc GraphQL schema

---
> “Một hệ thống tốt không chỉ deploy nhanh – mà rollback được cả khi thay đổi schema.”
