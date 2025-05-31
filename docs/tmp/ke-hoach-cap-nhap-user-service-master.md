**chuẩn hoá cấp độ 5⭐ toàn hệ thống**

---

## ✅ Kế hoạch cập nhật đồng bộ 4 file `user-service/master`

### 🎯 Mục tiêu:

Hoàn thiện tài liệu theo 4 trụ cột:
📐 **Thiết kế nghiệp vụ** – `design.md`
📊 **Mô hình dữ liệu** – `data-model.md`
🔗 **Giao tiếp & Hợp đồng API** – `interface-contract.md`
🧪 **Đặc tả kỹ thuật & Testable API** – `openapi.yaml`

---

### 📋 Danh sách cập nhật cụ thể theo từng file:

#### 1. `openapi.yaml` – ✅ **Trọng tâm chính**

| Hạng mục               | Việc cần làm                                                                      |
| ---------------------- | --------------------------------------------------------------------------------- |
| 🔘 Pagination & filter | Thêm `page`, `limit`, `search`, `sort` cho các `GET` danh sách                    |
| 🔘 RBAC metadata       | Thêm `x-required-permission` vào từng API                                         |
| 🔘 Audit tags          | Thêm `x-audit-action` hoặc `x-emits-event` để mô tả hành vi phát sinh log/sự kiện |
| 🔘 Event description   | Gắn metadata `emits: user.created` vào POST APIs                                  |
| 🔘 Schema chuẩn        | Đảm bảo mọi request/response dùng `$ref` (đã tốt rồi)                             |

---

#### 2. `interface-contract.md` – ✅ **Giao tiếp & Test**

| Hạng mục                     | Việc cần làm                                                       |
| ---------------------------- | ------------------------------------------------------------------ |
| 🟡 Bổ sung phần pagination   | Ghi rõ các `GET` có hỗ trợ `page`, `limit`, `search`, v.v.         |
| 🟡 Bổ sung kiểm thử contract | Mô tả cách test contract (vd: Postman, schemathesis, pytest-httpx) |
| 🟢 Mapping RBAC & Audit      | Gắn thông tin `permission required` + event trigger cho từng API   |
| 🟢 Cross-link với OpenAPI    | Link sang `openapi.yaml` để frontend/backend sync                  |

---

#### 3. `design.md` – ✅ **Luồng nghiệp vụ**

| Hạng mục                                    | Việc cần làm                                                     |
| ------------------------------------------- | ---------------------------------------------------------------- |
| 🟢 Mô tả pagination rõ ràng trong flow list | Nhấn mạnh pagination/filter có dùng cho admin tra cứu user       |
| 🟢 Mô tả sự kiện sinh ra                    | Khi tạo user/tenant → mô tả rõ có emit Pub/Sub event             |
| 🟡 Gắn RBAC & audit trace vào các flow      | Flow nào ghi log/audit nên thể hiện trong diagram hoặc chú thích |
| 🟡 Đề cập sơ lược về testability            | Gợi ý cách test các flow chính (mock Pub/Sub, kiểm tra audit)    |

---

#### 4. `data-model.md` – ✅ **Mô hình & quản lý dữ liệu**

| Hạng mục                 | Việc cần làm                                                                  |
| ------------------------ | ----------------------------------------------------------------------------- |
| 🟢 Pagination & indexing | Nhấn mạnh các bảng có pagination → cần index `created_at`, `tenant_id`        |
| 🟡 Audit logs            | Nếu audit là centralized → có thể note lại rằng audit lưu ở service khác      |
| 🟢 RBAC mapping logic    | Gắn chú thích cho `user_tenant_assignment` về policy + permission inheritance |

---

## 📌 Gợi ý quy trình làm

| Bước | Mô tả                          | Trách nhiệm                                                                |
| ---- | ------------------------------ | -------------------------------------------------------------------------- |
| 1️⃣  | Cập nhật `openapi.yaml`        | Tập trung vào 4 phần: pagination, RBAC, audit, emits                       |
| 2️⃣  | Update `interface-contract.md` | Mô tả test, gắn permission, event, audit                                   |
| 3️⃣  | Update `design.md`             | Diễn giải lại các flow chính (tạo user, gán tenant...) có thêm trace/audit |
| 4️⃣  | Update `data-model.md`         | Bổ sung note pagination index + luồng dữ liệu liên quan RBAC & log         |
| 5️⃣  | Review đồng bộ lần cuối        | Kiểm tra link chéo, schema, từ ngữ thống nhất                              |
| 6️⃣  | Ghi chú CHANGELOG.md           | Ghi rõ “Refactor v2 - chuẩn hoá 5⭐ API & tài liệu đồng bộ”                 |

---

## ⭐ **Checklist Chuẩn hóa `openapi.yaml` đạt 5⭐**

### ✅ 1. **Pagination & Filtering**

| ✅ Việc cần làm                                                                                   | Ghi chú                                                                      |
| ------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------- |
| \[ ] Thêm các query parameters `page`, `limit`, `search`, `sort` vào các API `GET` danh sách     | `/users-global`, `/tenants`, `/user-tenant-assignments`, `/rbac/templates/*` |
| \[ ] Sử dụng schema `Paginated[T]` với `meta.total`, `meta.page`, `meta.per_page` trong response | Nên định nghĩa 1 schema chung                                                |

---

### ✅ 2. **Phân quyền động (RBAC metadata)**

| ✅ Việc cần làm                                                                                       | Ghi chú                             |
| ---------------------------------------------------------------------------------------------------- | ----------------------------------- |
| \[ ] Thêm `x-required-permission` vào từng endpoint quan trọng (`GET`, `POST`, `PATCH`, `DELETE`)    | Phù hợp với `interface-contract.md` |
| \[ ] Permission code nên trùng với enum từ RBAC template (`user_global.read`, `tenant.create`, v.v.) | Dễ sinh doc/dev tool tự động hóa    |

---

### ✅ 3. **Gắn Audit Tags & Event Metadata**

| ✅ Việc cần làm                                                       | Ghi chú                                                                |
| -------------------------------------------------------------------- | ---------------------------------------------------------------------- |
| \[ ] Thêm `x-audit-action` vào API có thay đổi (POST, PATCH, DELETE) | Giúp log hoặc push audit message                                       |
| \[ ] Thêm `x-emits-event` vào API phát sinh Pub/Sub events           | Ví dụ: `user.created`, `tenant_user_assigned`, `rbac_template_updated` |

---

### ✅ 4. **Tối ưu Schema `$ref` & Type**

| ✅ Việc cần làm                                                                     | Ghi chú                       |
| ---------------------------------------------------------------------------------- | ----------------------------- |
| \[ ] Đảm bảo toàn bộ request/response dùng `$ref` không lặp inline                 | Giảm lỗi + auto-doc dễ hơn    |
| \[ ] Mỗi schema có `description`, `example`, `readOnly`/`writeOnly` nếu phù hợp    | Hỗ trợ codegen client tốt hơn |
| \[ ] Enum nên định nghĩa dưới `components/schemas` thay vì hardcode trong endpoint | Giúp reuse dễ hơn             |

---

### ✅ 5. **Bổ sung Tag Group rõ ràng**

| ✅ Việc cần làm                                                                            | Ghi chú                      |
| ----------------------------------------------------------------------------------------- | ---------------------------- |
| \[ ] Mỗi endpoint gắn tag như: `users-global`, `tenants`, `assignments`, `rbac-templates` |                              |
| \[ ] Khai báo mô tả tag ở phần đầu file (`tags:`)                                         | Dễ generate doc nhóm rõ ràng |

---

### ✅ 6. **Các cải tiến nâng cao**

| ✅ Việc cần làm                                                                               | Ghi chú                                  |
| -------------------------------------------------------------------------------------------- | ---------------------------------------- |
| \[ ] Thêm `nullable: true` cho các field optional                                            | Phù hợp với chuẩn Pydantic / OpenAPI 3.1 |
| \[ ] Thêm `oneOf` hoặc `allOf` cho các schema phức tạp (nếu có nhiều biến thể)               | Ví dụ `condition` của permission         |
| \[ ] Có schema `ErrorResponse` chuẩn hóa (`error.code`, `message`, `details`) và áp dụng đều | Tuân thủ ADR-011, 012                    |

---