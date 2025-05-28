# 📚 Service Catalog – Thiết kế Kiến trúc theo Service

Tài liệu này liệt kê các Service chính trong hệ thống `dx_vas`, được tổ chức theo định hướng microservices. Mỗi service có một thư mục riêng chứa các tài liệu thiết kế chi tiết.

## 🧱 Cấu trúc Tài liệu cho Mỗi Service

Mỗi thư mục của một service sẽ tuân theo cấu trúc chuẩn như sau:

```plaintext
docs/services/<service-name>/
├── design.md             # ✅ Tài liệu Thiết kế Tổng quan (Service Design Document - SDD)
├── interface-contract.md # 📘 Giao diện API mô tả bằng Markdown (business-level)
├── data-model.md         # 🗄️ Mô hình dữ liệu chi tiết – mô tả schema, quan hệ, chỉ mục
└── openapi.yaml          # 📡 Đặc tả kỹ thuật OpenAPI (machine-readable)
```

### 📄 Ý nghĩa của từng file:

* **`design.md`**
  Bao gồm: Scope, Trách nhiệm, Flow nghiệp vụ chính, Tương tác giữa service, Các sự kiện phát ra/nghe, Bảo mật, Cấu hình runtime, và Chiến lược test. Đây là tài liệu trung tâm của mỗi service.

* **`interface-contract.md`**
  Giao diện API mô tả ở mức nghiệp vụ (dễ đọc cho dev/backend/frontend). Dạng Markdown dễ review, không phụ thuộc YAML.

* **`data-model.md`**
  Mô tả các bảng, cột, quan hệ, constraint, index. Được viết rõ ràng để phục vụ review DB, code backend, và migration strategy.

* **`openapi.yaml`**
  Chuẩn hóa API specification cho CI/CD, test tự động, và dev tool (VSCode plugin, Swagger UI…).

---

✅ Việc tuân thủ cấu trúc trên giúp toàn bộ tài liệu kiến trúc của dx\_vas thống nhất, dễ tìm, dễ review, dễ onboard người mới.

---

## 🔝 Danh sách Services

| Thứ tự | Tên Service | Vai trò chính | Thư mục |
|--------|-------------|----------------|---------|
| 1️⃣ | User Service Master | Quản lý người dùng toàn cục, danh sách tenant, template RBAC | [`user-service/master/`](./user-service/master/) |
| 2️⃣ | Sub User Service (per tenant) | Quản lý user trong tenant, RBAC cục bộ, đồng bộ template | [`user-service/sub/`](./user-service/sub/) |
| 3️⃣ | Auth Service Master | Xác thực Google OAuth2, phát hành JWT toàn cục | [`auth-service/master/`](./auth-service/master/) |
| 4️⃣ | Sub Auth Service (per tenant) | Xác thực OTP/Local, phát hành JWT cục bộ | [`auth-service/sub/`](./auth-service/sub/) |
| 5️⃣ | API Gateway | Điểm vào duy nhất, định tuyến theo tenant, xác thực + RBAC | [`api-gateway/`](./api-gateway/) |
| 6️⃣ | Notification Service Master | Gửi thông báo toàn cục, publish Pub/Sub event | [`notification-service/master/`](./notification-service/master/) |
| 7️⃣ | Sub Notification Service (per tenant) | Gửi thông báo trong tenant, xử lý Pub/Sub + external channel | [`notification-service/sub/`](./notification-service/sub/) |
| 8️⃣ | CRM / SIS / LMS Adapter | Đồng bộ dữ liệu từ hệ thống ngoài (per tenant) | [`adapters/`](./adapters/) |
| 9️⃣ | Audit Logging Service (nếu tách riêng) | Ghi nhận và lưu trữ log hành vi toàn hệ thống | [`audit-log-service/`](./audit-log-service/) |

---

## 📂 Danh sách Service và Tài liệu Liên quan

> *Đang cập nhật theo tiến độ viết SDD từng service.*

Ví dụ:

### 🧠 User Service Master

- [Tổng quan thiết kế (design.md)](./user-service/master/design.md)
- [Giao diện API (interface-contract.md)](./user-service/master/interface-contract.md)
- [Mô hình dữ liệu (data-model.md)](./user-service/master/data-model.md)
- [OpenAPI Spec (openapi.yaml)](./user-service/master/openapi.yaml)

---
