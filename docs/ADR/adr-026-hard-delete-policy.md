---
id: adr-026-hard-delete-policy
title: ADR-026 - Chính sách Xóa Dữ liệu (Hard Delete) trong hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-05-30
tags: [data, retention, deletion, soft-delete, compliance, audit, dx-vas]
---

# ADR-026: Chính sách Xóa Dữ liệu (Hard Delete) trong hệ thống dx-vas

## 📌 Bối cảnh

Trong hệ thống dx-vas, việc xóa dữ liệu cần được kiểm soát chặt chẽ để:
- Đảm bảo tính truy vết (auditability)
- Tuân thủ quy định pháp lý (FERPA, GDPR, ...), chính sách ngành giáo dục
- Hỗ trợ khôi phục (recovery) và phân tích dữ liệu trong tương lai

Việc **xóa vật lý (hard delete)** thường không được khuyến khích nếu object có liên kết nghiệp vụ, quan hệ tham chiếu, hoặc ý nghĩa lịch sử. Vì vậy, hệ thống cần có một chiến lược rõ ràng để xác định **object nào KHÔNG nên hard delete**.

---

## 🧠 Tiêu chí Không Hard Delete

Một object **không nên hard delete** nếu thoả mãn một hoặc nhiều tiêu chí sau:

### 1. Có giá trị lịch sử hoặc cần thiết cho audit
- Ví dụ: `users_global`, `audit_logs`, `user_tenant_assignments`, điểm số, lịch sử học tập

### 2. Có quan hệ tham chiếu quan trọng (foreign key)
- Ví dụ: `users_global` ↔ `audit_logs`, `tenants` ↔ `user_tenant_assignments`, `roles` ↔ `user_roles`

### 3. Yêu cầu pháp lý hoặc compliance
- Ví dụ: Hồ sơ học sinh, giao dịch học phí, PII có retention riêng

### 4. Phục vụ phân tích hoặc khôi phục
- Ví dụ: CRM leads, khóa học cũ, hành vi người dùng

### 5. Có khả năng kích hoạt lại (reactivation)
- Ví dụ: Nhân sự quay lại, trường hoạt động trở lại, phân quyền được khôi phục

---

## ✅ Danh sách object KHÔNG được hard delete

| Object | Giải pháp thay thế hard delete |
|--------|-------------------------------|
| `users_global` | Trường `status`: `active` / `inactive` / `deleted` |
| `tenants` | Trường `status`: `active` / `suspended` / `archived` |
| `user_tenant_assignments` | Trường `assignment_status`: `active` / `revoked` |
| `roles_in_tenant` | Trường `status` hoặc `is_active` |
| `global_roles_templates` | Trường `status` hoặc `is_archived` |
| `permissions_in_tenant`, `global_permissions_templates` | Không xóa, chỉ deprecate code |
| `student_records` (SIS) | Trường `enrollment_status`, `is_archived` |
| Điểm số, giao dịch học phí | Không bao giờ xóa, có retention policy |
| `audit_logs` | Lưu trữ có thời hạn, không xóa thủ công |
| Leads (CRM) | Soft delete hoặc archive sau thời gian không hoạt động |

---

## 🧰 Cách triển khai Soft Delete / Archive

- Dùng trường `status`, `is_deleted`, `is_archived`, tùy ngữ cảnh
- Logic API phải filter các bản ghi đã bị soft delete
- Có thể thiết lập cronjob dọn dẹp (purge) theo retention policy rõ ràng

---

## ✅ Object có thể hard delete an toàn

| Object | Điều kiện |
|--------|----------|
| Temporary upload file | Hết hạn sau 7 ngày |
| OTP, mã xác minh | Hết hạn, không còn dùng |
| Draft form chưa submit | Sau 30 ngày hoặc khi user xoá |

---

## ✅ Lợi ích

- Bảo toàn dữ liệu có giá trị
- Hỗ trợ audit, rollback, reactivation
- Đáp ứng quy định compliance và phân tích nghiệp vụ

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Quá tải dữ liệu "đã xoá" | Có purge policy theo thời gian |
| Developer lỡ xoá nhầm | Bắt buộc soft delete mặc định qua API |
| Không thống nhất trạng thái | Xây dựng enum `status` chuẩn dùng chung toàn hệ thống |

---

## 📎 Tài liệu liên quan

- Data Retention & Privacy: [ADR-024](./adr-024-data-anonymization-retention.md)
- Audit Logging: [ADR-008](./adr-008-audit-logging.md)
- User Service: [ADR-006](./adr-006-auth-strategy.md), [ADR-007](./adr-007-rbac.md)
- Schema Migration: [ADR-023](./adr-023-schema-migration-strategy.md)

---

> “Dữ liệu không bao giờ chỉ là dữ liệu – nó là lịch sử, pháp lý, và sự thật cần được bảo vệ.”
