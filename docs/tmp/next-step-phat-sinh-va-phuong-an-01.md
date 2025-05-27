---
**Những việc cần làm tiếp theo trong thiết kế dựa trên các thống nhất này:**

1.  **Cập nhật `user-service/data-model.md`:**
    * Định nghĩa schema chi tiết cho User Service **Master**: các bảng `users_global` (với `auth_provider`, `local_auth_tenant_id`), `tenants`, `user_tenant_assignments`, `global_roles_templates`, `global_permissions_templates`.
    * Định nghĩa schema chi tiết cho Sub User Service (cấp **tenant**): các bảng `users_in_tenant` (tham chiếu `user_id_global`, có `is_active_in_tenant`), `roles_in_tenant`, `permissions_in_tenant`, và các bảng mapping RBAC cục bộ. (Phần này bạn đã làm khá tốt rồi, chỉ cần rà soát lại cho khớp với vai trò của Master).
2.  **Cập nhật `user-service/design.md`:**
    * Làm rõ vai trò và trách nhiệm riêng biệt của User Service Master và Sub User Service (tenant-level).
    * Vẽ/mô tả các luồng tương tác mới:
        * Luồng Superadmin gán user vào tenant.
        * Luồng Admin tenant gán role cho user trong tenant.
        * Luồng đồng bộ user từ Master xuống Sub User Service qua Pub/Sub.
        * Luồng User Service Master cung cấp dữ liệu RBAC cho API Gateway (dựa trên `user_id` và `tenant_id` trong JWT).
3.  **Cập nhật `user-service/interface-contract.md` và `user-service/openapi.yaml`:**
    * Định nghĩa các API cho User Service **Master** (ví dụ: API quản lý tenant, API gán user vào tenant, API quản lý template role/permission).
    * Rà soát lại API của Sub User Service (tenant-level) để đảm bảo nó chỉ xử lý trong phạm vi tenant của mình và tương tác đúng với User Service Master nếu cần.
4.  **Cập nhật Thiết kế cho Auth Service (Master và Sub):**
    * Tạo `design.md`, `interface-contract.md`, `openapi.yaml`, `data-model.md` (nếu Sub Auth Service có state) cho cả Auth Service Master và Sub Auth Service (hoặc một tài liệu Auth Service chung mô tả cả hai vai trò).
    * Đặc tả rõ luồng cấp phát JWT "đầy đủ".
5.  **Cập nhật `system-diagrams.md`:**
    * Sơ đồ tổng quan cần phản ánh rõ sự tồn tại của User Service Master và các Sub User Service (hoặc cách các stack của tenant tương tác với Master).
    * Có thể cần thêm sơ đồ kiến trúc multi-tenant chi tiết hơn.
6.  **Cập nhật `README.md` (Kiến trúc tổng thể) và `rbac-deep-dive.md`:**
    * Đảm bảo các tài liệu này phản ánh đúng mô hình quản lý RBAC tập trung kết hợp với tùy chỉnh ở cấp tenant.

Đây là một khối lượng công việc cập nhật đáng kể, nhưng nó sẽ đảm bảo rằng toàn bộ thiết kế của chúng ta nhất quán và sẵn sàng cho việc triển khai một hệ thống multi-tenant mạnh mẽ và linh hoạt.