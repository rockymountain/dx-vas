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