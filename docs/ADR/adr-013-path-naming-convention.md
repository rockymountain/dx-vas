---
id: adr-013-path-naming-convention
title: ADR-013: Quy ước đặt tên endpoint, method và tham số cho API hệ thống dx-vas
status: accepted
author: DX VAS Architecture Team
date: 2025-06-22
tags: [api, naming, path, rest, dx-vas]
---

## 📌 Bối cảnh

Trong hệ thống **dx-vas**, nhiều API được phát triển bởi các đội khác nhau. Một số inconsistency phổ biến đã xảy ra:
- Endpoint dạng `/getAllStudents`, `/updateCourse`, không tuân theo chuẩn REST
- Tham số có lúc là `snake_case`, lúc lại `camelCase`
- Đôi khi action được encode vào tên path thay vì dùng HTTP method đúng cách

Điều này gây khó khăn cho:
- Người dùng API (frontend, mobile, service khác)
- Việc sinh tài liệu tự động từ OpenAPI
- Khả năng phân tích logs và theo dõi endpoint

## 🧠 Quyết định

**Áp dụng chuẩn RESTful + đặt tên nhất quán cho path, method và tham số API toàn hệ thống, tuân theo quy ước cụ thể sau.**

## 📏 Quy ước đặt tên path

### ✅ Tên tài nguyên dùng **số nhiều**, `snake_case`, tiếng Anh
- Ví dụ: `/students`, `/courses`, `/teachers`
- Nếu truy cập 1 phần tử: `/students/{student_id}`

### ✅ Dùng tiền tố version: `/api/v1/...`

### ✅ Không encode hành động vào tên path
- ❌ Không dùng: `/createStudent`, `/getAllCourses`, `/deleteUser`
- ✅ Thay bằng:
  - `POST /students`
  - `GET /courses`
  - `DELETE /users/{id}`

### ✅ Hành động đặc biệt dùng sub-path dạng động từ
- `POST /students/{id}/deactivate`
- `POST /classes/{id}/assign`
- Luôn document rõ action này là “custom”

### ✅ Tham số path dùng `snake_case`
- `GET /students/{student_id}` (not `studentId` or `StudentId`)

## ⚙️ Quy ước HTTP Method

| Method | Mục đích | Ví dụ |
|--------|----------|-------|
| `GET` | Lấy dữ liệu | `GET /students`, `GET /students/{id}` |
| `POST` | Tạo mới | `POST /students` |
| `PUT` | Ghi đè toàn bộ | `PUT /students/{id}` |
| `PATCH` | Cập nhật một phần | `PATCH /students/{id}` |
| `DELETE` | Xoá | `DELETE /students/{id}` |

## 🔍 Quy ước tham số truy vấn (query param)

### ✅ Tên tham số `snake_case`, rõ nghĩa
- `?class_id=5A&page=2&limit=20`

### ✅ Paging: sử dụng `page`, `limit`
- `page`: số trang bắt đầu từ **1**
- `limit`: số phần tử tối đa mỗi trang
- Ví dụ: `?page=2&limit=10` → lấy phần tử thứ 11 đến 20

### ✅ Sắp xếp: dùng `sort_by`, `order`
- `sort_by`: tên field để sắp xếp (ví dụ: `created_at`, `score`)
- `order`: hướng sắp xếp, nhận giá trị `asc` hoặc `desc` (mặc định là `asc` nếu không có)
- **Quy ước áp dụng khi sắp xếp nhiều trường:**
  - Dùng tiền tố `-` trước tên field để chỉ `desc`, không có tiền tố là `asc`
  - Ví dụ: `?sort_by=-created_at,name` → sắp xếp `created_at DESC, name ASC`

## 🧪 Kiểm tra tự động
- Tích hợp linter vào CI/CD để bắt endpoint sai định dạng
- Lint rule cho OpenAPI generator để enforce path/method/param đúng

## ✅ Lợi ích

- Giảm chi phí học API cho dev mới, client
- Tăng khả năng sinh docs, test, mock tự động
- Thống nhất khi log và monitor endpoint API

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Các API legacy không theo chuẩn | Có layer chuyển đổi tạm (alias route), hoặc bổ sung docs cảnh báo |
| Một số action đặc biệt không thể biểu diễn bằng REST | Cho phép dùng `/resource/{id}/action` dạng POST, nhưng phải document rõ |

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Đặt tên tuỳ ý theo hành động (get, create...) | Vi phạm chuẩn REST, khó sinh tool/docs |
| Dùng camelCase trong path/params | Không thống nhất với phần còn lại (đa số codebase snake_case) |

## 📎 Tài liệu liên quan

- API Error format: [ADR-011](./adr-011-api-error-format.md)
- API Governance: [ADR-009](./adr-009-api-governance.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)

---
> "Một API tốt được nhận biết không chỉ qua dữ liệu nó trả – mà qua cách nó đặt tên."
