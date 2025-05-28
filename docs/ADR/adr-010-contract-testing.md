---
id: adr-010-contract-testing
title: ADR-010 - Chiến lược Contract Testing cho hệ thống dx-vas
status: accepted
author: DX VAS Architecture Team
date: 2025-05-22
tags: [contract-testing, pact, integration, api, dx-vas]
---

# ADR-010: Contract Testing giữa các dịch vụ

## Bối cảnh

Hệ thống dx-vas bao gồm nhiều service giao tiếp qua HTTP hoặc Pub/Sub. Việc đảm bảo các service tương tác đúng với nhau (hợp đồng API không bị phá vỡ khi cập nhật) là rất quan trọng, đặc biệt trong mô hình microservices đa tenant.

Contract testing sẽ giúp phát hiện sớm lỗi tương thích khi có thay đổi schema, đầu vào/đầu ra, hoặc khi các team phát triển độc lập.

## Quyết định

### 1. Áp dụng hình thức contract testing

- Sử dụng mô hình **Consumer-Driven Contract Testing**
- Tools đề xuất: **Pact**, **Dredd**, hoặc custom test runners
- Các consumer (Gateway, Frontend, Service khác) định nghĩa expectation → producer (Sub Service) cam kết đáp ứng

### 2. Contract testing phải hỗ trợ tenant context

Với mô hình multi-tenant, mọi lời gọi service đều có ngữ cảnh `tenant_id`, `user_id`, `roles`, v.v.

Do đó, contract test **cần bao gồm**:

- Header: `x-tenant-id`, `authorization (Bearer <jwt>)`
- Payload có chứa: `user_id`, hoặc thông tin RBAC liên quan
- Response phù hợp với dữ liệu trong ngữ cảnh tenant cụ thể

### 3. Các nhóm service cần testing

| Service | Consumer | Mục tiêu contract |
|---------|----------|-------------------|
| Sub User Service | Auth Service / Gateway | Đảm bảo cung cấp đúng `roles`, `permissions` cho `user_id`, `tenant_id` |
| Sub Notification Service | Notification Master | Đảm bảo nhận sự kiện `global_notification_requested` và phản hồi đúng |
| Sub Auth Service | Frontend / Gateway | Đảm bảo login OTP trả JWT đúng định dạng, context |
| SIS Adapter | Sub User Service | Trả thông tin học sinh đúng RBAC/tenant context |

### 4. Kết nối với CI/CD

- Mỗi PR vào service phải chạy contract test liên quan
- Nếu hợp đồng bị thay đổi → yêu cầu phê duyệt từ consumer tương ứng
- Contracts được lưu ở repo riêng hoặc cùng repo backend (trong thư mục `/contracts`)

## Hệ quả

✅ Ưu điểm:

- Giảm thiểu lỗi không tương thích giữa các dịch vụ
- Bảo vệ luồng xác thực và phân quyền đa tenant
- Phát hiện sớm sai khác nếu service thay đổi (VD: schema, auth claim, error format)

⚠️ Lưu ý:

- Contract test cần dữ liệu mẫu theo từng `tenant_id`
- JWT test nên dùng fixture/generator thay vì token thật
- Một số service sử dụng Pub/Sub cần thêm test listener giả lập (hoặc sử dụng test harness)

## Liên kết liên quan

- [`adr-006-auth-strategy.md`](./adr-006-auth-strategy.md)
- [`adr-007-rbac.md`](./adr-007-rbac.md)
- [`README.md#15-testing--contract`](../README.md#15-testing--contract)

---
> “Contract testing là cách để các dịch vụ giao tiếp bằng sự tin cậy – chứ không phải niềm tin mù quáng.”
