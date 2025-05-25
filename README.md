# Dự án Chuyển đổi số Trường Việt Anh (dx-vas)

Chào mừng bạn đến với dự án dx-vas! Đây là hệ thống chuyển đổi số toàn diện được phát triển cho Trường Quốc tế Việt Anh (VAS), nhằm hiện đại hóa và tối ưu hóa các quy trình vận hành, quản lý và tương tác trong nhà trường.

## 🎯 Mục tiêu dự án

Dự án dx-vas hướng tới việc xây dựng một nền tảng tích hợp, bao gồm các chức năng chính như:

* Quản lý thông tin học sinh, giáo viên, và phụ huynh.
* Quản lý lớp học, lịch học, điểm danh.
* Quản lý học phí và các khoản thu.
* Hệ thống thông báo đa kênh.
* Cổng thông tin cho phụ huynh và học sinh (Customer Portal - PWA).
* Ứng dụng quản trị cho nhân viên và giáo viên (Admin Webapp - SPA).
* Tích hợp với các hệ thống hiện có như CRM (SuiteCRM), SIS (Gibbon), và LMS (Moodle).
* Quy trình tuyển sinh trực tuyến.

## 🚀 Công nghệ sử dụng (Tổng quan)

* **Backend:** Microservices phát triển bằng Python (FastAPI), triển khai trên Google Cloud Run.
* **Frontend:**
    * Customer Portal: Progressive Web App (PWA) - (React/Vue/Angular - *tùy chọn của bạn*)
    * Admin Webapp: Single Page Application (SPA) - (React/Vue/Angular - *tùy chọn của bạn*)
* **Cơ sở dữ liệu:**
    * PostgreSQL (cho Core Services) - triển khai trên Google Cloud SQL.
    * MySQL (cho các Adapters tích hợp hệ thống legacy) - triển khai trên Google Cloud SQL.
* **Caching:** Redis (Google Cloud Memorystore for Redis).
* **Nhắn tin & Sự kiện:** Google Cloud Pub/Sub.
* **Lưu trữ file:** Google Cloud Storage (GCS).
* **API Gateway:** Quản lý request, xác thực, phân quyền (RBAC).
* **Xác thực:** Google OAuth2, OTP.
* **CI/CD:** GitHub Actions / Google Cloud Build.
* **Hạ tầng dưới dạng mã (IaC):** Terraform.

## 📚 Tài liệu chi tiết

Toàn bộ tài liệu thiết kế kiến trúc, sơ đồ hệ thống, hướng dẫn phát triển (Dev Guide), và các quyết định kiến trúc (ADRs) đều được tập trung tại thư mục `docs`.

👉 **Để bắt đầu, vui lòng truy cập vào trang chỉ mục chính của tài liệu tại đây:** [**docs/index.md**](./docs/index.md)

Trang chỉ mục này sẽ cung cấp các liên kết đến:

* [Tài liệu Kiến trúc Tổng quan](./docs/README.md).
* [Chi tiết Kiến trúc Đăng nhập & Phân quyền Động (RBAC)](./docs/architecture/rbac-deep-dive.md).
* [Tổng hợp Sơ đồ Kiến trúc Hệ thống (bao gồm sơ đồ tổng quan, các luồng chi tiết, sơ đồ triển khai)](./docs/architecture/system-diagrams.md).
* [Danh sách các Quyết định Kiến trúc](./docs/ADR/index.md).
* [Hướng dẫn dành cho Nhà phát triển](./docs/dev/dev-guide.md).

## 🤝 Đóng góp

Chúng tôi luôn chào đón sự đóng góp! Vui lòng xem qua [hướng dẫn đóng góp (CONTRIBUTING)](./docs/dev/CONTRIBUTING.md) để biết thêm chi tiết về quy trình làm việc, coding convention và cách tạo Pull Request.

## 📝 Giấy phép

© Trường Việt Anh – DX VAS Project 2025. Mọi quyền được bảo lưu. Mã nguồn dành riêng cho mục đích nội bộ và đào tạo.

---

> Made with ❤️ by the Legendary Dev Team @ VAS