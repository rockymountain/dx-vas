---
id: adr-010-contract-testing
title: ADR-010: Chiến lược Contract Testing cho hệ thống dx-vas
status: accepted
author: DX VAS Architecture Team
date: 2025-06-22
tags: [contract-testing, pact, integration, api, dx-vas]
---

## 📌 Bối cảnh

Trong hệ thống **dx-vas**, nhiều dịch vụ độc lập giao tiếp với nhau qua API:
- Frontend (Portal/Admin) ↔ API Gateway
- API Gateway ↔ LMS Adapter, CRM Adapter, Notification Service
- API Gateway ↔ External systems (SSO, tuyển sinh...)

Các dịch vụ được phát triển bởi các đội khác nhau, thay đổi độc lập. Điều này dẫn đến nguy cơ **breaking change** nếu một producer thay đổi API mà consumer chưa kịp thích nghi. Contract Testing là giải pháp đảm bảo sự tương thích đó.

## 🧠 Quyết định

**Áp dụng Consumer-Driven Contract Testing bằng công cụ Pact, với hỗ trợ Pact Broker để quản lý contracts giữa các dịch vụ trong dx-vas. Tích hợp contract testing vào CI/CD cả phía consumer và producer.**

## 📖 Khái niệm chính

- **Producer**: Dịch vụ cung cấp API (Gateway, Adapter)
- **Consumer**: Dịch vụ gọi API (Frontend, API Gateway gọi backend...)
- **Contract**: Một tệp JSON định nghĩa kỳ vọng của consumer với response từ producer

## 🧩 Cặp producer–consumer áp dụng contract testing

| Producer | Consumer |
|----------|----------|
| API Gateway | Frontend (Portal, Admin Webapp) |
| LMS Adapter | API Gateway |
| CRM Adapter | API Gateway |
| Notification Service | API Gateway |
| Public API | Hệ thống tuyển sinh (external consumer) |

## 🔄 Quy trình làm việc

### 1. Consumer side
- Viết test giả lập gọi API của producer bằng `pact-js`, `pact-python`, `pact-go`...
- Sinh Pact file (JSON) thể hiện request/response kỳ vọng
- Gửi Pact file lên Pact Broker (hoặc commit vào repo producer nếu chưa có Broker)

### 2. Producer side
- Cài provider verifier (`pact-provider-verifier`)
- Lấy pact từ Broker/repo → chạy test thực tế
- Đảm bảo API thực tế đáp ứng đúng contract của consumer

### 3. CI/CD tích hợp
- Consumer CI:
  - Tạo Pact file sau mỗi build
  - Upload Pact file lên Broker (hoặc Git)
- Producer CI:
  - Tự động verify contract với mỗi commit
  - Fail nếu có breaking change (khác contract)

### 4. Pact Broker
- Lưu trữ Pact theo phiên bản consumer
- Theo dõi tương thích giữa các phiên bản API
- Hỗ trợ webhook → tự động trigger CI bên producer khi consumer cập nhật contract

## 🧪 Provider States
- Cho phép producer set up data phù hợp trước khi verify một interaction
- Được định nghĩa bởi consumer trong contract
- Producer mapping các state → data setup tương ứng trong test

## 📌 Áp dụng trong dx-vas

- Bắt buộc contract test với tất cả API public hoặc shared
- Là một bước trong checklist review OpenAPI (xem [`adr-009-api-governance.md`](./adr-009-api-governance.md))
- Pact file phải được duyệt nếu là breaking change → thêm tag `breaking` vào commit/pull request
- Producer được phép từ chối contract không hợp lệ hoặc chưa hỗ trợ

## 🛠 Công cụ đề xuất

| Mục tiêu | Công cụ |
|---------|---------|
| Viết contract (consumer) | `pact-js`, `pact-python`, `pact-go`, `pact-net` |
| Verify contract (producer) | `pact-provider-verifier` |
| Broker | Pactflow (SaaS) hoặc self-hosted Pact Broker |
| CI/CD | GitHub Actions, GitLab CI, webhook từ Broker |

## ✅ Lợi ích

- Phát hiện breaking change trước khi deploy
- Tăng độ tin cậy và khả năng mở rộng giữa các service
- Đảm bảo frontend/backend phát triển độc lập nhưng nhất quán
- Giảm lỗi runtime do thiếu đồng bộ API

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Pact file cũ/lỗi bị verify sai | Tích hợp schema validator + version tag |
| Không có test cho interaction edge case | Bắt buộc coverage cho API critical |
| Producer không muốn verify mọi contract | Giới hạn contract scope theo `provider_tag` |

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Manual integration test giữa các service | Không scale, chậm, khó tự động hoá |
| Chỉ dùng test frontend để phát hiện API bug | Không bảo vệ producer trước thay đổi silent |

## 📎 Tài liệu liên quan

- API Governance: [ADR-009](./adr-009-api-governance.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)

---
> “Contract testing là cách để các dịch vụ giao tiếp bằng sự tin cậy – chứ không phải niềm tin mù quáng.”
