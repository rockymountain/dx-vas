Tuyệt vời – chúng ta hoàn toàn có đủ dữ liệu để xây dựng một **Data Management Strategy (Chiến lược Quản lý Dữ liệu)** cho hệ thống **dx-vas**. Dưới đây là **bản đề xuất đầy đủ**, tổng hợp và chuẩn hóa từ các ADR đã có (`ADR-003`, `ADR-005`, `ADR-023`, `ADR-024`, `ADR-026`, v.v.):

---

## 📚 **DX-VAS Data Management Strategy**

### 🎯 Mục tiêu

Đảm bảo dữ liệu trong hệ thống dx-vas được:

* **Lưu trữ đúng quy định**
* **Xử lý an toàn, truy vết được**
* **Xóa, ẩn danh, lưu trữ** theo chiến lược rõ ràng
* **Tối ưu hóa chi phí và hiệu suất**

---

## 1. 📦 **Phân loại dữ liệu**

| Loại dữ liệu                | Mô tả                       | Ví dụ                                 |
| --------------------------- | --------------------------- | ------------------------------------- |
| **PII (Thông tin cá nhân)** | Có thể định danh người dùng | Họ tên, email, SĐT, IP                |
| **Dữ liệu nghiệp vụ**       | Quản lý vận hành hệ thống   | Điểm số, học phí, user-role           |
| **Dữ liệu tạm thời**        | Phục vụ xử lý trung gian    | OTP, draft form                       |
| **Log & Trace**             | Ghi nhận hành vi hệ thống   | Audit logs, request logs              |
| **Analytics**               | Phân tích thống kê, BI      | Số lượng học sinh đăng ký, tỷ lệ lead |

---

## 2. 🔐 **Anonymization & Privacy**

Theo [ADR-024](./adr-024-data-anonymization-retention.md):

* Mọi PII đều phải được **mask/hash/ẩn danh** khi dùng ngoài production (dev/staging/demo)
* Logs phải được filter qua middleware trước khi gửi ra GCP/Datadog
* Email, IP, tên học sinh cần được xử lý theo quy định FERPA/GDPR nếu xuất ra ngoài

---

## 3. ♻️ **Soft Delete, Archive & Retention**

Theo [ADR-026](./adr-026-hard-delete-policy.md) & [ADR-023](./adr-023-schema-migration-strategy.md):

### ❗ Không được hard delete:

* `users_global`, `tenants`, `user_tenant_assignments`
* `audit_logs`, điểm số, học phí, lịch sử học
* `roles`, `permissions`, global templates

→ Dùng `status`, `is_deleted`, `is_archived`, `enrollment_status`...

### 🧹 Retention policy đề xuất:

| Loại dữ liệu         | Thời gian lưu   | Ghi chú                       |
| -------------------- | --------------- | ----------------------------- |
| Logs (info/debug)    | 30 ngày         | Có thể archive thêm 6 tháng   |
| Audit logs           | 180 ngày (prod) | Không xoá thủ công            |
| PII đã ngừng sử dụng | 365 ngày        | Sau đó pseudonymize/anonymize |
| Refresh token        | 30–90 ngày      | Theo loại user                |
| OTP / Draft          | 1–7 ngày        | Có thể hard delete            |

---

## 4. 🔁 **Schema Migration & Compatibility**

Theo [ADR-023](./adr-023-schema-migration-strategy.md):

* **Không DROP/RENAME trực tiếp**
* Migration phải forward-compatible, rollbackable
* Dùng shadow field và transition flags
* Migration script chạy tách biệt deploy (qua CI/CD)

---

## 5. 🛡️ **Bảo mật dữ liệu (Secrets & Access)**

Theo [ADR-003](./adr-003-secrets.md), [ADR-004](./adr-004-security.md):

* Secrets lưu trữ trong Google Secret Manager, không hardcoded
* Truy cập dữ liệu production bị giới hạn (RBAC + audit trail)
* Mọi export dữ liệu phải ghi log + có lý do (incident, research)

---

## 6. 📊 **Observability & Chi phí dữ liệu**

Theo [ADR-020](./adr-020-cost-observability.md):

* Gắn label `dx-vas_service`, `env`, `team` cho mọi resource
* Dùng alert nếu Redis, Cloud Logging, Cloud Run vượt ngưỡng
* Export billing → BigQuery → dashboard Looker Studio

---

## 7. 🧭 **Quy trình tiêu chuẩn liên quan đến dữ liệu**

| Hoạt động                    | Yêu cầu                          | Tài liệu liên quan |
| ---------------------------- | -------------------------------- | ------------------ |
| Xoá người dùng               | Soft delete + Retention 365 ngày | ADR-026, ADR-024   |
| Tạo schema mới               | Qua migration 3 bước             | ADR-023            |
| Sử dụng dữ liệu thật để test | Phải chạy `anonymize_dataset.py` | ADR-024            |
| Truy cập dữ liệu nhạy cảm    | Có audit trail                   | ADR-008            |
| Export dữ liệu               | Có lý do, ghi log                | ADR-004, ADR-024   |

---

## 🧩 Mở rộng đề xuất (tùy theo giai đoạn tiếp theo)

* Tạo dashboard quản lý trạng thái soft delete theo thời gian
* Thiết lập **Purge Service** tự động dọn dữ liệu soft-delete > X ngày
* Tạo bộ linter rule cho `log PII`, `missing status field`, `unsafe delete` trong CI/CD

---

## 📎 Tài liệu liên quan

* [ADR-003 - Secrets](./adr-003-secrets.md)
* [ADR-004 - Security](./adr-004-security.md)
* [ADR-005 - Env Config](./adr-005-env-config.md)
* [ADR-023 - Schema Migration](./adr-023-schema-migration-strategy.md)
* [ADR-024 - Data Anonymization & Retention](./adr-024-data-anonymization-retention.md)
* [ADR-026 - Hard Delete Policy](./adr-026-hard-delete-policy.md)

---

Bạn muốn tôi chuyển phần này thành `adr-027-data-management-strategy.md` không? Hay bạn muốn tích hợp vào tài liệu thiết kế hệ thống tổng thể (`README.md`)?
