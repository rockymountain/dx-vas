---
id: adr-024-data-anonymization-retention
title: ADR-024 - Chính sách ẩn danh hóa và lưu trữ dữ liệu cho hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-05-22
tags: [privacy, anonymization, retention, data-governance, dx-vas]
---

## 📌 Bối cảnh

Hệ thống **dx-vas** lưu trữ và xử lý nhiều loại dữ liệu có khả năng nhận dạng cá nhân (PII):
- Thông tin học sinh, phụ huynh, giáo viên (họ tên, số điện thoại, email)
- Nhật ký hành vi người dùng (logs, trace)
- Thông tin cấu hình tài khoản, lịch sử thao tác hệ thống

Ngoài ra, hệ thống còn cần sao lưu, tạo môi trường thử nghiệm (sandbox/dev), và phân tích sự kiện phục vụ cải tiến. Nếu không kiểm soát tốt:
- Dữ liệu nhạy cảm có thể bị rò rỉ ra môi trường không bảo mật
- Log có thể chứa thông tin PII không cần thiết
- Thông tin cá nhân có thể bị lưu trữ quá lâu mà không có lý do hợp lệ

---

## 🧠 Quyết định

**Thiết lập chính sách phân loại, ẩn danh hóa và lưu trữ dữ liệu chuẩn hoá trên toàn hệ thống dx-vas để đảm bảo an toàn, tuân thủ, và dễ dàng áp dụng sandbox hoặc audit.**

---

## 🧩 Phân loại dữ liệu

| Loại dữ liệu | Ví dụ | Phân loại |
|--------------|-------|-----------|
| PII | Họ tên, email, SĐT, IP | Nhạy cảm – cần ẩn danh hoặc hạn chế truy cập |
| Technical log | API path, trace ID, latency | Có thể chứa PII nếu bao gồm IP đầy đủ – cần kiểm soát và lọc |
| Audit trail | Ai làm gì, lúc nào | Nhạy cảm mức vừa – cần giữ nguyên nhưng bảo vệ truy cập |
| Business metrics | số học sinh đăng ký, doanh thu | Tổng hợp, không cần ẩn danh |

---

## 🔐 Chính sách ẩn danh hóa (Anonymization)

| Trường | Phương pháp |
|--------|-------------|
| Email, tên | Mask (`n***@gmail.com`), randomize, fake data |
| IP address | Hash (SHA256) hoặc cắt bớt (e.g., /24 subnet) – coi là PII nếu đầy đủ |
| Mã học sinh | Tạo mapping tạm thời (UUID ↔ student_id) |
| Logs chứa PII | Dùng lớp middleware/log filter để lọc/mask trước khi ghi. **Không log trực tiếp PII dù ở mức debug/info.** |

> Tất cả log phải đi qua lớp filter trước khi gửi ra ngoài (GCP Logging, Datadog...)

---

## 📦 Sandbox & Test Environment

- Clone dữ liệu thật → chạy script `anonymize_dataset.py`
- Dữ liệu sau khi ẩn danh:
  - Không còn truy ngược học sinh thực tế
  - Không chứa thông tin liên lạc thật
  - Có thể dùng trong dev/test/demo/public QA

- Mọi môi trường ngoài production bắt buộc dùng dữ liệu đã ẩn danh

---

## 🗂 Chính sách lưu trữ dữ liệu (Retention)

| Loại dữ liệu | Retention | Ghi chú |
|--------------|-----------|--------|
| Logs hệ thống | 30 ngày (default) | Có thể archive 6 tháng với BigQuery |
| Audit logs | 180 ngày (prod), 30 ngày (dev) | Tuân thủ audit nội bộ |
| Dữ liệu học sinh không còn hoạt động | 365 ngày | Tuân thủ theo thông tư ngành giáo dục về lưu trữ hồ sơ học sinh |
| Access token | 15 phút | Theo [`adr-006`](./adr-006-auth-strategy.md) – dùng cho phiên hoạt động |
| Refresh token | 30–90 ngày tùy loại user | Theo [`adr-006`](./adr-006-auth-strategy.md) – lưu trong Redis hoặc encrypted store |

> Tất cả retention sẽ được kiểm soát qua lifecycle rule (Cloud Storage), TTL (Firestore/Redis), hoặc job cleanup tự động (batch script)

---

## 🛡️ Bảo mật & Quyền truy cập

- Chỉ nhóm Platform/Data có quyền truy cập dữ liệu gốc
- Mọi export dữ liệu phải:
  - Được ghi nhận trong audit log
  - Có lý do cụ thể (debug, incident, product research)
  - Nếu chứa PII → phải xử lý ẩn danh trước khi chia sẻ

---

## ✅ Lợi ích

- Đảm bảo compliance nếu áp dụng chính sách như GDPR, FERPA
- Hạn chế rủi ro lộ dữ liệu qua log, demo, testing
- Dễ dàng tạo sandbox để huấn luyện AI, kiểm thử sản phẩm

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Developer vô tình log thông tin nhạy cảm | Filter log + CI/CD linter check log string |
| Dữ liệu thật lọt ra môi trường staging | Anonymize bắt buộc + kiểm tra sau deploy |
| Retention config bị bỏ quên | Có cron job + dashboard kiểm tra TTL, storage usage |

---

## 📎 Tài liệu liên quan

- Secrets: [ADR-003](./adr-003-secrets.md)
- Security:[ADR-004](./adr-004-security.md)
- Observability: [ADR-005](./adr-005-observability.md)
- External Observability: [ADR-021](./adr-021-external-observability.md)
- Audit Logging: [ADR-008](./adr-008-audit-logging.md)
- Schema Migration: [ADR-023](./adr-023-schema-migration-strategy.md)
- Auth Strategy: [ADR-006](./adr-006-auth-strategy.md)

---
> “Dữ liệu thật rất giá trị – và càng phải được bảo vệ nghiêm ngặt hơn khi dùng sai mục đích.”
