**Với Mỗi Service Được Ưu tiên, Thực Hiện Các Bước Thiết Kế Chi Tiết Sau:**
    * **a. Định Nghĩa Rõ Phạm Vi và Trách Nhiệm (Scope & Responsibilities):**
        * Mặc dù đã có ở mức tổng quan, giờ là lúc đào sâu: Service này chịu trách nhiệm chính xác cho những nghiệp vụ nào? Nó quản lý những thực thể dữ liệu nào? Giới hạn của nó là gì?
        * **Tài liệu tham khảo:** `README.md` (phần mô tả service), `system-diagrams.md` (vai trò trong các luồng).
    * **b. Thiết Kế API Chi Tiết (Interface Contract):**
        * Định nghĩa tất cả các API endpoint mà service này sẽ cung cấp (public API qua Gateway) hoặc tiêu thụ (nếu gọi service khác).
        * Sử dụng OpenAPI (Swagger) để đặc tả chi tiết: path, HTTP methods, request parameters (path, query, header, body), response schemas (success, error), và các mã lỗi có thể có.
        * Áp dụng các quy ước đã định nghĩa trong `backend-dev-guide.md` (Mục 6: API design).
        * **Hành động:** Tạo/cập nhật file OpenAPI spec cho service đó trong thư mục `docs/interfaces/`.
    * **c. Thiết Kế Mô Hình Dữ Liệu Chi Tiết (Detailed Data Model):**
        * Xác định các bảng CSDL cần thiết cho service (nếu có stateful).
        * Thiết kế schema chi tiết cho từng bảng: tên cột, kiểu dữ liệu, ràng buộc (PK, FK, UNIQUE, NOT NULL, CHECK), và các chỉ mục (indexes).
        * Cân nhắc việc sử dụng PostgreSQL hay MySQL dựa trên quyết định của chúng ta (Core Services dùng PostgreSQL, Adapters dùng MySQL).
        * **Hành động:** Có thể tạo một file Markdown riêng cho data model của service đó (ví dụ: `docs/services/user-service/data-model.md`) hoặc bổ sung vào một phần thiết kế chi tiết của service.
    * **d. Đặc Tả Các Luồng Xử Lý Nghiệp Vụ Chính (Detailed Business Logic Flows):**
        * Với các use case phức tạp bên trong service, vẽ sơ đồ luồng chi tiết hơn (activity diagram, sequence diagram) để mô tả các bước xử lý, các quyết định rẽ nhánh, và tương tác với repository hoặc các service khác (nếu có, qua Gateway).
        * Tham khảo các quy ước trong `backend-dev-guide.md` (Mục 5: Xử lý nghiệp vụ & usecase mẫu).
    * **e. Xác Định Các Sự Kiện (Events) mà Service Sẽ Phát Ra hoặc Lắng Nghe:**
        * Nếu service tham gia vào các luồng event-driven (qua Pub/Sub).
        * Định nghĩa rõ tên topic, cấu trúc payload của sự kiện.
        * Xem xét các nguyên tắc idempotency đã có trong `backend-dev-guide.md` (Mục 9).
    * **f. Cân Nhắc về Bảo Mật và Phân Quyền ở Cấp Service:**
        * Service này cần những permission nào để hoạt động? (để khai báo trong file `permissions.yaml` mà `backend-dev-guide.md` có đề cập).
        * Cách service kiểm tra các header phân quyền (`X-Permissions`) từ Gateway như thế nào?
    * **g. Cân Nhắc về Cấu Hình và Dependencies:**
        * Liệt kê các biến môi trường, secrets mà service cần.
        * Các service khác mà nó phụ thuộc vào.
    * **h. Chiến Lược Testing Cụ Thể cho Service:**
        * Xác định các kịch bản unit test và integration test quan trọng dựa trên nghiệp vụ của service.
        * Tham khảo `backend-dev-guide.md` (Mục 10).