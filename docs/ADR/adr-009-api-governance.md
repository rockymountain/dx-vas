---
id: adr-009-api-governance
title: ADR-009: Chính sách Quản trị API (API Governance) cho hệ thống dx-vas
status: accepted
author: DX VAS Architecture Team
date: 2025-06-22
tags: [api, governance, versioning, standards, dx-vas]
---

## 📌 Bối cảnh

Hệ thống **dx-vas** bao gồm nhiều dịch vụ cung cấp và tiêu thụ API:
- API Gateway (điểm vào trung tâm)
- Các backend: CRM Adapter, LMS Proxy, Notification Service, SIS Adapter...
- Frontend webapp (Portal, Admin)
- Dịch vụ bên ngoài (Zalo, Google OAuth, Firebase...)

Để đảm bảo sự **thống nhất, bảo trì lâu dài và khả năng mở rộng**, toàn hệ thống cần có **chính sách quản trị API (API Governance)** chung.

## 🧠 Quyết định

**Áp dụng chính sách API Governance chuẩn, bao gồm versioning, linting, schema hóa OpenAPI, quy trình review, và tài liệu hoá tự động. Áp dụng cho tất cả các API nội bộ và external-facing trong dx-vas.**

## 📐 Nguyên tắc chung

### 1. API phải có version rõ ràng
- Sử dụng prefix trong path: `/api/v1/...`
- Thay đổi breaking → tạo version mới (`v2`)
- Các version song song được duy trì trong thời gian chuyển tiếp (6–12 tháng)

### 2. API phải có tài liệu OpenAPI
- Viết OpenAPI schema (YAML/JSON) theo chuẩn 3.x
- Mỗi service cần expose `GET /openapi.json`
- Dùng Swagger UI hoặc Redoc để render trong CI hoặc Docs

### 3. Linting & kiểm tra tự động
- Dùng `speccy`, `spectral`, hoặc `oas-linter` để:
  - Bắt lỗi thiếu description, type
  - Kiểm tra naming convention, security scheme, 2xx/4xx đầy đủ
- Tích hợp trong CI/CD:
  - Fail build nếu OpenAPI có lỗi nặng

### 4. Tên endpoint và schema có quy tắc
- Tên endpoint RESTful: `/students/{id}`, `/grades/{student_id}`
- Tham số đường dẫn `snake_case`, danh từ số nhiều
- Dữ liệu trả về dạng JSON, response chuẩn hoá `{data, error, meta}`

### 5. Versioning & Deprecation Policy
- Mỗi version API được ghi rõ ngày phát hành và ngày dự kiến ngưng hỗ trợ (sunset date)
- Endpoint cũ được thêm header:
  ```http
  Deprecation: true
  Sunset: Wed, 31 Dec 2025 23:59:59 GMT
  Link: </api/v2/...>; rel="successor-version"
  ```
- Có endpoint `/api/docs/deprecation` để liệt kê API sắp ngưng hỗ trợ

### 6. Quy trình review và phê duyệt
- Mỗi thay đổi schema cần mở Pull Request với:
  - File OpenAPI mới hoặc diff
  - Ghi rõ breaking / non-breaking
  - Chạy pass linter + test contract (xem [ADR-010](./adr-010-contract-testing.md))
- Review bởi team kiến trúc và đại diện frontend/backend liên quan

### 7. Gắn metadata cho mỗi API
- Thêm tag, owner, description trong OpenAPI
- Mục tiêu: giúp team phân tích và hiểu rõ từng API endpoint

```yaml
x-metadata:
  owner: lms_team
  tier: critical
  service: lms_proxy
```

## 🛠 Công cụ đề xuất

| Mục tiêu | Công cụ |
|---------|---------|
| Lint & kiểm tra schema | Spectral, Speccy, oasdiff |
| So sánh version | `oasdiff`, `schemathesis diff` |
| Render docs | Redoc, Swagger UI, Stoplight |
| CI/CD tích hợp | GitHub Actions (`lint-openapi.yml`) |

## ✅ Lợi ích

- API dễ hiểu, dễ tích hợp và mở rộng
- Giảm lỗi khi tích hợp đa dịch vụ / frontend
- Tăng khả năng tuân thủ, kiểm soát thay đổi API
- Giúp các team làm việc trơn tru hơn nhờ tài liệu tự động và thống nhất

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| API không có schema rõ ràng → lỗi tích hợp | Bắt buộc check trong CI/CD, fail nếu thiếu OpenAPI |
| Không quản lý version tốt → phá vỡ tích hợp | Gắn version, áp dụng chính sách deprecation rõ ràng |
| Review bị chậm | Có checklist PR + phân quyền người duyệt theo module |

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Không dùng version prefix | Dễ gây breaking change ngầm |
| Chỉ viết tài liệu Markdown | Không tự sinh schema, khó check tự động |
| Quản lý API tự do theo từng team | Thiếu chuẩn hoá, khó maintain hệ thống lớn |

## 📎 Tài liệu liên quan

- Contract Testing: [ADR-010](./adr-010-contract-testing.md)
- Deployment Strategy: [ADR-010](./adr-011-zero-downtime.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)

---
> “API tốt là tài liệu – nhưng API có governance tốt là tài sản của cả tổ chức.”
