Đúng rồi – để xây dựng được **`data-model.md` tổng hợp**, bạn cần chuẩn bị **Software Structure Diagrams (SSD)** hoặc ít nhất là **schema chi tiết cho từng service** trước. Dưới đây là lộ trình hợp lý:

---

## ✅ Bước chuẩn bị trước khi tạo `data-model.md`

### 1. 📊 Tạo SSD / Data Schema cho từng service

Bạn nên phác thảo hoặc liệt kê rõ cấu trúc bảng và quan hệ cho các service sau:

| Service                         | File SSD cần có       | Ghi chú                                                                   |
| ------------------------------- | --------------------- | ------------------------------------------------------------------------- |
| ✅ **User Service Master**       | `ssd-user-master.md`  | `users_global`, `tenants`, `user_tenant_assignments`, role/perm templates |
| ✅ **Sub User Service**          | `ssd-user-sub.md`     | `users_in_tenant`, `roles`, `permissions`, `mapping`, `user_status`       |
| ✅ **Auth Service (Master/Sub)** | `ssd-auth.md`         | OTP tracking, login audit, refresh token (nếu có)                         |
| ✅ **Notification Service**      | `ssd-notification.md` | `notification_request`, `channel_config`, `delivery_log`, Pub/Sub status  |
| 🔄 **Adapter (CRM/SIS)**        | `ssd-adapter.md`      | Sync học sinh, giáo viên từ hệ thống ngoài                                |
| ✅ **Audit Logging Service**     | `ssd-audit-log.md`    | `audit_logs` theo chuẩn `actor`, `action`, `target`                       |

---

## ✅ Sau khi có SSD → Mới tạo `data-model.md`

File `data-model.md` sẽ là bản tổng hợp có cấu trúc như:

```markdown
# 🧩 Data Model Tổng Hợp

## 1. User Service Master
### 1.1 Bảng `users_global`
...

## 2. Sub User Service
...

## 3. Notification Service
...

## 4. Quan hệ giữa các bảng liên tenant
- `users_global.user_id` → `users_in_tenant.user_id_global`
- ...
```

---

## 👇 Bạn muốn bắt đầu từ SSD service nào trước?

Mình khuyên bắt đầu từ:

1. `ssd-user-master.md`
2. `ssd-user-sub.md`

→ Sau đó mới đến Notification và Auth, vì 2 cái đầu đóng vai trò trụ cột trong RBAC và xác định định danh tenant.

Chỉ cần bạn xác nhận, mình có thể bắt đầu với MARKDOWN BLOCK cho từng SSD theo chuẩn rõ ràng, có cả sơ đồ nếu cần. Bạn chọn service nào trước?
