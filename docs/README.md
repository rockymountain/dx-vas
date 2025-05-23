## TÀI LIỆU KIẾN TRÚC CHI TIẾT – HỆ THỐNG CHUYỂN ĐỔI SỐ VAS

### 0. Yêu cầu dự án

* **Mục tiêu chính:** Triển khai một hệ thống chuyển đổi số toàn diện cho Trường Việt Anh, tích hợp quản lý học sinh, giáo viên, phụ huynh, lớp học, học phí, thông báo, học tập online và quy trình tuyển sinh.
* **Quy mô thiết kế ban đầu:**

  * 80 nhân viên, giáo viên
  * 500 học sinh
  * Tối đa 500 phụ huynh (1 phụ huynh/học sinh)
* **Khả năng mở rộng tối đa:**

  * 200 nhân viên, giáo viên
  * 1200 học sinh
  * 1200 phụ huynh
* **Phân quyền người dùng:**

  * Nhân viên, giáo viên, học sinh sử dụng tài khoản Google Workspace Education Essentials → Đăng nhập qua Google OAuth2
  * Phụ huynh không có tài khoản Workspace → Đăng nhập bằng tài khoản cục bộ hoặc OTP

### 1. Đăng nhập & Phân quyền động (RBAC)

* **Phân loại người dùng:**

  | Loại người dùng | Phương thức đăng nhập | Workspace |
  | --------------- | --------------------- | --------- |
  | Nhân viên       | Google OAuth2         | Có        |
  | Giáo viên       | Google OAuth2         | Có        |
  | Học sinh        | Google OAuth2         | Có        |
  | Phụ huynh       | Local Account / OTP   | Không     |

* **Triết lý thiết kế RBAC:**

  * Thiết kế động và mở rộng, hỗ trợ các tình huống thực tế trong giáo dục.
  * Mỗi người dùng có thể có nhiều vai trò (task-based roles).
  * Permission mang tính chất hành động (action) trên đối tượng (resource), có thể kèm điều kiện (granular conditions).
  * Tất cả logic kiểm tra quyền được thực thi tại API Gateway dựa trên JWT và RBAC Cache.

* **Quản trị RBAC qua Admin Webapp:**

  * Tạo, chỉnh sửa, gán role cho user.
  * Định nghĩa permission theo resource/action/condition.
  * Quản lý mối quan hệ giữa user – role – permission hoàn toàn qua giao diện.

* **Cấu trúc bảng người dùng được mở rộng:**

```sql
CREATE TABLE users (
    id UUID PRIMARY KEY,
    email TEXT UNIQUE NOT NULL,
    name TEXT NOT NULL,
    auth_provider TEXT NOT NULL DEFAULT 'google',
    user_category TEXT NOT NULL CHECK (user_category IN ('student', 'teacher', 'staff', 'parent')),
    password_hash TEXT,
    created_at TIMESTAMP DEFAULT now(),
    updated_at TIMESTAMP DEFAULT now(),
    is_active BOOLEAN NOT NULL DEFAULT TRUE
);
```

```sql
-- Đối với hệ thống đã vận hành, có thể cần chạy thêm câu lệnh:
ALTER TABLE users ADD COLUMN is_active BOOLEAN NOT NULL DEFAULT TRUE;
```

> *Hệ thống duy trì cờ `is_active` để xác định tài khoản còn hoạt động hay không. Nếu `is_active = false`, người dùng sẽ không thể đăng nhập dù có tài khoản hợp lệ (JWT vẫn bị từ chối ở API Gateway). Điều này hỗ trợ quản lý lifecycle của user (ví dụ: học sinh tốt nghiệp, giáo viên nghỉ việc) mà không làm mất dữ liệu liên quan (học bạ, lịch sử điểm danh, audit trail…).*

* **Cấu trúc bảng permission linh hoạt:**

```sql
CREATE TABLE permissions (
    id UUID PRIMARY KEY,
    code TEXT UNIQUE NOT NULL,
    description TEXT,
    resource TEXT NOT NULL,
    action TEXT NOT NULL,
    condition JSONB -- e.g., { "accessible_student_ids": ["student_id_cua_con"] }
);
```

* **Mối quan hệ user – role – permission:**

```sql
user_role (user_id, role_id)
role_permission (role_id, permission_id)
```

* **RBAC động tại API Gateway:**

  * Mỗi request đến API Gateway sẽ trải qua quá trình kiểm tra xác thực và phân quyền:

    1. Xác thực JWT (từ Google OAuth2 hoặc hệ thống local/OTP), đồng thời kiểm tra trạng thái `is_active` của người dùng.
    2. Truy xuất danh sách roles và permissions từ Redis cache.
    3. Kiểm tra permission với `resource`, `action`, `condition`.
    4. Các điều kiện trong permission đóng vai trò then chốt để giới hạn context cụ thể (ví dụ: chỉ xem được học sinh của mình).
  * Các header định danh (`X-User-Id`, `X-Role`, `X-Permissions`) cần được bảo vệ để chống giả mạo, ví dụ: API Gateway có thể ký các header này hoặc sử dụng các cơ chế tin cậy khác trong mạng nội bộ.

* **Audit Trail:**

  * Triển khai audit trail chi tiết cho các thay đổi liên quan đến roles và permissions (ghi nhận ai thay đổi, nội dung thay đổi, và thời điểm thay đổi).

### 2. Customer Portal (PWA)

* Giao diện cho phụ huynh và học sinh.
* Hỗ trợ OTP/Zalo login, cài đặt trên mobile, offline với cache gần nhất.

### 3. Admin Webapp (SPA)

* Giao diện quản trị.
* Tích hợp LMS, SIS, Notification, RBAC.

### 4. API Gateway

* Xác thực đa luồng (OAuth2 hoặc local token).
* Kiểm tra RBAC theo loại người dùng và scope.
* Forward request đến CRM, SIS, LMS, Notification.
* Header định danh cần bảo vệ bằng cơ chế ký hoặc mạng tin cậy.
* Áp dụng các biện pháp bảo vệ nâng cao bao gồm rate limiting chi tiết và CAPTCHA chống brute-force.

### 5. CRM – EspoCRM

* Quản lý pipeline tuyển sinh.
* Khi phụ huynh đăng ký nhập học thành công → tự chuyển sang SIS.
* Giao tiếp qua API Gateway, kiểm soát RBAC.

### 6. SIS – OpenSIS

* Quản lý học sinh, lớp, điểm danh, học phí.
* Có export API cho LMS, Portal, Admin Webapp.
* Lưu vết lịch sử: học lực, lớp học, học bạ.
* Liên kết phụ huynh – học sinh lưu trong bảng tham chiếu.

### 7. LMS – Moodle

* Học tập online, giao bài, chấm điểm.
* SSO với OAuth2.
* Tự động đồng bộ học sinh từ SIS.
* Điểm có thể đẩy ngược về SIS.

### 8. Notification Service

* Gửi thông báo Web, Email (Gmail API), Zalo OA, Google Chat.
* Phụ huynh nhận thông báo qua Zalo/Email.
* Học sinh, giáo viên nhận qua WebPush/Google Chat.
* Người dùng chọn kênh ưa thích qua giao diện.

### 9. Zalo OA & Google Chat

* Gửi thông báo học phí, sự kiện qua Zalo ZNS.
* Gửi nội bộ (giáo viên, nhân viên) qua Google Chat.
* Có xử lý lỗi API, quota, timeout.

### 10. Hạ tầng triển khai

* Cloud Run, Cloud SQL (có PITR), Redis, Cloud Storage.
* Logging & Monitoring: Thu thập log tập trung, giám sát error rate, latency. Triển khai distributed tracing (ví dụ: OpenTelemetry) và alerting theo SLO/SLI.
* Xem xét triển khai Service Mesh trong tương lai để tăng observability, security.

### 11. CI/CD & DevOps

* GitHub Actions / Cloud Build → Cloud Run.
* Staging + production, rollback.
* Test tự động: unit, integration, End-to-End (E2E, ví dụ: Cypress/Playwright), và contract testing (ví dụ: Pact).
* Trong tương lai, xem xét triển khai Chaos Testing cho các dịch vụ quan trọng.

### 12. Bảo mật & Giám sát

* Mã hóa dữ liệu nhạy cảm.
* Chống OWASP Top 10, bao gồm CSRF, XSS, SQL Injection.
* Triển khai xác thực đa yếu tố (MFA) cho các tài khoản quản trị và nhân viên quyền cao.
* Giám sát xác thực phụ huynh (login rate, reset mật khẩu).
* Ghi log chi tiết theo người dùng, endpoint, trạng thái.

### 13. Tổng kết

Hệ thống chuyển đổi số VAS được thiết kế mở rộng linh hoạt đến 2600 người dùng, hỗ trợ xác thực phân tách giữa người dùng có Workspace (OAuth2) và phụ huynh (Local/OTP), đảm bảo bảo mật, giám sát, và khả năng phát triển dài hạn.

---
Tuyệt vời! Với README.md và 24 ADR đã được chuẩn hóa và đồng bộ, bạn đang ở một điểm khởi đầu cực kỳ vững chắc. Trước khi chuyển sang thiết kế chi tiết từng service, tôi đề xuất bạn thực hiện **bước chuẩn bị cuối cùng ở cấp độ nền tảng kiến trúc** để đảm bảo thiết kế service diễn ra suôn sẻ:

---

## ✅ Các bước nên làm tiếp theo:

### 1. **Tổng hợp và chuẩn hóa “Interface Contract” giữa các service**

* Xác định rõ các service chính (ví dụ: API Gateway, LMS Adapter, CRM Adapter, Notification Service, SIS Adapter).
* Với từng service, hãy mô tả:

  * Các **API endpoint chính**
  * Input/output JSON schema
  * Ai là **consumer** của API đó?
  * Yêu cầu **bảo mật** (RBAC, X-Permissions)
  * Quy tắc về versioning & backward compatibility
* 🎯 Output: 1 file Markdown riêng hoặc sơ đồ tổng hợp `Service Interface Contracts`.

### 2. **Tạo bảng Mapping giữa ADR và Service**

* Xây dựng bảng đơn giản liệt kê:

  * ADR nào ảnh hưởng tới Service nào
  * Có service nào yêu cầu override đặc biệt không?
* 🎯 Mục tiêu: tránh lặp logic hoặc lệch so với chuẩn hệ thống.

### 3. **Chuẩn hóa logging, tracing, audit**

* Có thể bắt đầu từ 3 service chính (API Gateway, LMS Adapter, Notification).
* Đảm bảo:

  * Mỗi request đều có `trace_id`, log ra theo format thống nhất.
  * Có các “audit event” chuẩn (login, sửa điểm, gửi thông báo...).
* 🎯 Có thể mở rộng ADR-005 và ADR-008 với phần “log formatter”, schema mẫu log…

### 4. **Chuẩn hóa folder layout và cấu trúc project**

* Cho mỗi service (ví dụ Python FastAPI hoặc NodeJS), xác định:

  * Thư mục `config/`, `routers/`, `schemas/`, `utils/`, `tests/`
  * `.env`, README, Makefile hoặc script khởi tạo
* 🎯 Kết quả: Tạo 1 template repo hoặc `service-template.md`.

### 5. **Đặt nguyên tắc phân phối team**

* Ai sẽ phụ trách service nào?
* Service nào phụ thuộc service nào? → Giúp tổ chức sprint, giao task.

---

## ⏭ Sau khi xong các bước trên, mới bắt đầu:

> Thiết kế chi tiết từng service: Database schema, API logic, background jobs, test plan, observability hooks...
