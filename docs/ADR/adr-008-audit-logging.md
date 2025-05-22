---
id: adr-008-audit-logging
title: ADR-008: Chiến lược Audit Logging cho hệ thống dx_vas
status: accepted
author: DX VAS Security & Platform Team
date: 2025-06-22
tags: [audit, logging, observability, dx_vas, security]
---

## 📌 Bối cảnh

Hệ thống **dx_vas** xử lý dữ liệu nhạy cảm và hành vi người dùng:
- Quản lý thông tin học sinh, giáo viên, điểm số (SIS, LMS)
- Tác vụ phân quyền, đăng nhập, cập nhật hồ sơ (CRM, Gateway)
- Gửi thông báo, gọi API tích hợp (Notification, External Services)

Để phục vụ kiểm toán (audit), bảo mật, truy vết hành vi và điều tra sự cố, cần hệ thống **Audit Logging tập trung và chuẩn hoá** trên toàn bộ các service.

## 🧠 Quyết định

**Áp dụng chiến lược Audit Logging chuẩn hoá, gửi log dạng JSON tới Cloud Logging, phân loại theo `audit_level`, lưu dài hạn và giới hạn truy cập. Áp dụng cho cả Gateway và các backend service.**

## 🧩 Cấu trúc log audit chuẩn

```json
{
  "timestamp": "2025-06-22T14:00:00Z",
  "request_id": "abc-123",
  "user_id": "u_567",
  "role": "admin",
  "ip": "203.113.1.5",
  "action": "update_student",
  "resource": "student/102",
  "method": "PUT",
  "status_code": 200,
  "latency_ms": 147,
  "source": "gateway|crm_adapter|lms_proxy",
  "actor_type": "human|service|system",
  "audit_level": "critical|info|debug"
}
```

- `resource`: dạng `{type}/{id}`
- `source`: định danh service
- `audit_level`:
  - `critical`: thao tác ghi nhạy cảm (PUT/DELETE/role change)
  - `info`: xem dữ liệu quan trọng (GET student/grades)
  - `debug`: mặc định off, chỉ dùng dev/test

## 🔄 Luồng tích hợp

### Tại API Gateway
- Log mọi hành động cần phân quyền → log sau khi xác thực và kiểm tra RBAC
- Viết middleware `audit_logger.py`
- Gửi log qua stdout → Cloud Logging

### Tại Backend Services
- Cung cấp SDK hoặc decorator `@audit_event`
- Gọi `audit_logger.log(...)` tại các API có side-effect
- Có thể gửi về chung 1 topic Pub/Sub hoặc stdout riêng của từng service

## 📦 Lưu trữ & Truy vấn

- Dữ liệu log lưu tại Cloud Logging (180 ngày)
- Export sang BigQuery → tạo bảng `audit_logs_dxvas`
- Query theo:
  - `user_id`, `action`, `role`, `source`, `audit_level`
  - Thống kê theo ngày/tháng/quý để phục vụ audit nội bộ

## 🔐 Kiểm soát truy cập & bảo mật

- Chỉ nhóm `Platform Admin` và `Security Team` được xem full audit log
- Phân quyền chi tiết theo log type: `system`, `user`, `security`
- Ghi log hành vi đọc audit log nếu cần (meta-audit)
- Mask thông tin nhạy cảm: không log full payload, chỉ log ID hoặc field xác định

## 🛠 Tích hợp CI & Kiểm thử

- Kiểm thử middleware tạo log đúng schema
- Kiểm tra CI reject nếu thiếu audit log cho API ghi nhạy cảm
- Có thể mock `audit_logger.log()` trong unit test

## ✅ Lợi ích

- Tăng khả năng kiểm soát hành vi hệ thống
- Hỗ trợ điều tra sự cố, phản ứng nhanh khi bị tấn công
- Đáp ứng yêu cầu kiểm toán nội bộ, tuân thủ dữ liệu
- Dễ thống kê hoạt động quan trọng theo người dùng và thời gian

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Ghi log quá mức gây overload | Chỉ log hành động rõ ràng trong danh sách cho phép |
| Log chứa dữ liệu nhạy cảm | Mask hoặc log field định danh thay vì full object |
| Bỏ sót hành động nhạy cảm | CI/CD kiểm tra + checklist tại PR + middleware bắt buộc |

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Chỉ log lỗi hệ thống | Không đủ để audit hành vi người dùng |
| Log chung lẫn access/error log | Khó tách, phân quyền, và phân tích |
| Log tại backend riêng biệt, không đồng bộ schema | Không query được tập trung, thiếu chuẩn hóa |

## 📎 Tài liệu liên quan

- RBAC Strategy: [ADR-007](./adr-007-rbac.md)
- Auth Strategy: [ADR-006](./adr-006-auth-strategy.md)
- Security Hardening: [ADR-004](./adr-004-security.md)

---
> “Nếu bạn không ghi lại hành vi của hệ thống – bạn sẽ không bao giờ biết điều gì đã xảy ra khi nó xảy ra.”
