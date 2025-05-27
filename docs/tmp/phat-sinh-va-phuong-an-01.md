## Tổng Hợp Vấn đề Phát Sinh và Hướng Giải Quyết Đã Thống Nhất (Multi-Tenancy)

### 1. Yêu cầu Cốt Lõi và Bối Cảnh Mới:

* **Multi-Tenancy:** Hỗ trợ nhiều trường thành viên.
* **Cách Ly Nghiêm Ngặt:** Mỗi trường có dữ liệu, quyền, cấu hình, và **Frontend App riêng**.
* **Hệ thống Legacy Riêng Lẻ:** Mỗi trường có instance CRM/SIS/LMS riêng.
* **Quản Lý User Linh Hoạt:** Nhân viên có thể có nhiều vai trò ở nhiều trường. Cần cơ chế xin cấp quyền.
* **Quản Trị Tập Trung:** Superadmin quản lý toàn hệ thống (user, tenant, role, permission), có khả năng tổng hợp báo cáo.
* **Google Workspace Chung:** Dùng chung cho nhân viên/giáo viên/học sinh (ở các trường dùng Workspace).
* **Học sinh dùng Local/OTP Login:** Ở một số trường (ví dụ: trung tâm tiếng Anh, mầm non), học sinh sẽ dùng tài khoản Local/OTP.
* **Định hướng Tập Trung Dữ Liệu:** Cho RPA và AI trong tương lai.

### 2. Giải Pháp Kiến Trúc Đã Thống Nhất:

* **Mô Hình Triển Khai "Separate Stacks per Tenant" (Deployment Cách 1):**
    * **Giải pháp:** Mỗi trường thành viên (tenant) sẽ có một bộ Frontend Apps (PWA, SPA) riêng, Business Adapters (CRM, SIS, LMS) riêng, và Sub Auth Service riêng (để quản lý Local/OTP login của tenant đó).
    * **Lý do:** Đảm bảo cách ly hoàn toàn dữ liệu, quyền, cấu hình, giao diện và xử lý xác thực local cho từng trường. Phù hợp với việc mỗi trường có instance CRM/SIS/LMS riêng.

* **API Gateway Chung (Shared API Gateway):**
    * **Giải pháp:** Vẫn sử dụng một API Gateway chung.
    * **Xác định Tenant:** API Gateway sẽ dựa vào `tenant_id` trong JWT (sau khi người dùng đã chọn trường/được xác thực cho một trường cụ thể) để định tuyến request đến backend stack của tenant tương ứng, hoặc đến các API của Tenant Master.
    * **Lưu ý:** Domain của Frontend App riêng của từng trường có thể giúp API Gateway trong việc xác định ngữ cảnh ban đầu của request.

* **User Service Master Quản lý Tất Cả Định Danh User:**
    * **Giải pháp:** Sẽ có một **User Service "Master"** duy nhất, là nơi quản lý tập trung và là nguồn chân lý cho **tất cả định danh người dùng** trên toàn hệ thống dx_vas.
        * Bao gồm cả user Google Workspace và user Local/OTP từ các tenant.
        * Mỗi user sẽ có một `user_id` toàn cục duy nhất.
        * Bảng `users` trong User Service Master sẽ có các trường như `auth_provider` ('google', 'local_otp') và `local_auth_tenant_id` (để biết user local đó thuộc tenant nào).
    * User Service Master cũng quản lý danh sách các `tenants`, mối quan hệ `user_tenant_assignments` (user nào được gán/thuộc về tenant nào), và các `global_roles_templates`, `global_permissions_templates`.
    * **Sub User Service (cấp tenant) sẽ quản lý RBAC cụ thể cho tenant đó:**
        * Quản lý `roles_in_tenant`, `permissions_in_tenant` (có thể clone/tùy chỉnh từ template của Master hoặc tạo mới).
        * Quản lý các bảng mapping RBAC cục bộ (`user_role_in_tenant`, `role_permission_in_tenant`).
        * Lưu trữ tham chiếu đến `user_id_global` và cờ `is_active_in_tenant`.

* **Đồng bộ/Tham chiếu User từ Master xuống Sub User Service:**
    * **Giải pháp (Option B):** User Service Master sẽ phát sự kiện qua Pub/Sub (ví dụ: `user_assigned_to_tenant` hoặc `user_info_updated_for_tenant`) khi Superadmin gán user vào tenant hoặc khi có cập nhật thông tin user toàn cục.
    * Sub User Service của tenant liên quan sẽ lắng nghe và tạo/cập nhật bản ghi "tham chiếu" hoặc "shadow user" cục bộ.

* **Trạng thái `is_active` của User:**
    * **Giải pháp:** User Service Master quản lý trạng thái `is_active` toàn cục. Mỗi Sub User Service sẽ có thêm cờ `is_active_in_tenant` riêng để quản lý trạng thái hoạt động của user *trong phạm vi tenant đó*.

* **Auth Service và Luồng Cấp Phát JWT "Đầy Đủ":**
    * **Auth Service Master:** Xử lý chính luồng Google OAuth2. Sau khi Google xác thực, và sau khi người dùng chọn tenant (nếu cần), Auth Service Master sẽ phối hợp với User Service Master để lấy thông tin (user có thuộc tenant đó không) và có thể là cả với Sub User Service của tenant (để lấy role cụ thể trong tenant) trước khi phát hành JWT "đầy Đủ" (chứa `user_id_global`, `tenant_id`, và `role(s)/permission(s)` trong tenant).
    * **Sub Auth Service của Tenant:** Chịu trách nhiệm chính cho việc **xác thực Local/OTP** cho người dùng của tenant đó (bao gồm phụ huynh và học sinh ở các trường không dùng Workspace). Sau khi xác thực local thành công:
        * Sub Auth Service sẽ gọi lên User Service Master để đăng ký người dùng này (nếu là lần đầu) và nhận về `user_id` toàn cục.
        * Sau đó, Sub Auth Service sẽ phối hợp với Sub User Service của tenant mình để lấy thông tin RBAC (roles/permissions) của user local đó *trong tenant đó* và phát hành JWT "đầy đủ".

* **Superadmin Webapp Chung:**
    * Sử dụng "Tenant Master" để quản lý tập trung user, tenant, template role/permission, và cấu hình kết nối cho từng tenant.
    * Có khả năng tổng hợp báo cáo (yêu cầu thiết kế Data Warehouse/Data Lake và quy trình ETL từ dữ liệu của các tenant).

* **Global/Template Roles & Permissions:**
    * User Service Master sẽ quản lý một bộ "Global/Template Roles & Permissions" mà các tenant có thể sử dụng, kế thừa, hoặc tùy chỉnh tại Sub User Service của mình.

