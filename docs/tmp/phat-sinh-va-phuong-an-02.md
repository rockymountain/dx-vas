Trong khi tập trung vào User Service và Auth Service cho kiến trúc multi-tenant, chúng ta đã chưa đào sâu về việc **Notification Service** sẽ hoạt động như thế nào trong bối cảnh mới này.

Đây là một điểm rất quan trọng, vì mỗi trường thành viên (tenant) có thể sẽ có nhu cầu về nội dung thông báo, kênh gửi, và đối tượng nhận thông báo riêng.

Hãy cùng phân tích và đưa ra hướng giải quyết cho Notification Service trong kiến trúc multi-tenant, dựa trên những gì chúng ta đã thống nhất:

---

## Phân tích và Đề xuất cho Notification Service trong Kiến trúc Multi-Tenant

**Yêu cầu đối với Notification Service trong môi trường Multi-Tenant:**

* **Nội dung thông báo theo từng trường:** Template email, nội dung ZNS, tin nhắn Google Chat có thể cần tùy chỉnh cho từng trường.
* **Kênh gửi riêng của từng trường:** Mỗi trường có thể có Zalo OA riêng, cấu hình gửi email riêng (ví dụ: từ địa chỉ email của trường đó), webhook Google Chat riêng.
* **Đối tượng nhận thông báo thuộc từng trường:** Thông báo học phí của trường A chỉ gửi cho phụ huynh/học sinh của trường A.
* **Log và tracking riêng biệt:** Mỗi trường có thể muốn theo dõi lịch sử gửi thông báo của riêng mình.
* **Có thể có thông báo toàn hệ thống:** Superadmin có thể muốn gửi thông báo đến tất cả các trường hoặc một nhóm trường.

**Giải pháp kiến trúc đề xuất cho Notification Service:**

Dựa trên mô hình **"Separate Stacks per Tenant"** và việc có một **"Tenant Master"**, chúng ta có thể áp dụng một mô hình tương tự cho Notification Service:

1.  **Sub Notification Service (Notification Service ở cấp độ Tenant):**
    * **Trách nhiệm chính:**
        * Quản lý template thông báo riêng cho tenant đó.
        * Lưu trữ cấu hình kênh gửi riêng của tenant (Zalo OA API keys, Gmail API credentials/service account, Google Chat webhooks của tenant đó).
        * Xử lý các yêu cầu gửi thông báo phát sinh từ các service nghiệp vụ (CRM Adapter, SIS Adapter, LMS Adapter) hoặc Core Services "shadow" (nếu có) của tenant đó.
        * Gọi đến các API của nhà cung cấp dịch vụ (Zalo, Gmail, Google Chat) sử dụng credentials của tenant.
        * Ghi log và theo dõi trạng thái gửi thông báo cho tenant đó.
    * **Triển khai:** Mỗi tenant sẽ có một instance Notification Service riêng, thuộc "stack riêng" của họ.
    * **API:** Cung cấp API nội bộ (ví dụ: `POST /tenant-notifications`) để các service khác trong cùng stack của tenant có thể gọi (thông qua API Gateway của tenant đó hoặc trực tiếp nếu trong cùng mạng nội bộ được kiểm soát).

2.  **Notification Service Master (Hoặc chức năng trong một Core Service của Tenant "Master"):**
    * **Trách nhiệm chính:**
        * Xử lý các yêu cầu gửi thông báo mang tính chất **toàn hệ thống** từ Superadmin Webapp (ví dụ: thông báo bảo trì hệ thống đến tất cả quản trị viên trường).
        * Có thể quản lý các **template thông báo chung** mà các Sub Notification Service có thể kế thừa hoặc sử dụng.
        * **Điều phối gửi thông báo đến nhiều tenant:** Nếu Superadmin muốn gửi một thông báo đến một nhóm trường hoặc tất cả các trường, Notification Service Master sẽ nhận yêu cầu này. Sau đó, nó có thể:
            * **Option A (Master gọi Sub):** Gọi lần lượt API của từng Sub Notification Service (của các tenant đích) để yêu cầu họ gửi thông báo (sử dụng template và kênh của tenant đó).
            * **Option B (Master phát sự kiện):** Phát một sự kiện "global notification request" lên Pub/Sub, các Sub Notification Service lắng nghe và tự xử lý nếu thông báo đó áp dụng cho tenant của họ.
    * **API:** Cung cấp API cho Superadmin Webapp (ví dụ: `POST /global-notifications`).

**Luồng hoạt động:**

* **Thông báo trong phạm vi Tenant (Ví dụ: Trường A gửi thông báo học phí):**
    1.  SIS Adapter của Trường A phát hiện học sinh X cần đóng học phí.
    2.  SIS Adapter của Trường A gọi API của **Sub Notification Service của Trường A** (qua API Gateway, đã được định tuyến đến stack của Trường A).
    3.  Sub Notification Service của Trường A lấy template thông báo học phí của Trường A, thông tin liên hệ của phụ huynh học sinh X (từ Sub User Service của Trường A hoặc User Service Master + `tenant_id`), và credentials kênh gửi (ví dụ Zalo OA của Trường A).
    4.  Sub Notification Service của Trường A gửi thông báo và ghi log.

* **Thông báo Toàn Hệ Thống (Ví dụ: Superadmin gửi thông báo cập nhật chính sách):**
    1.  Superadmin soạn thông báo trên Superadmin Webapp.
    2.  Superadmin Webapp gọi API của **Notification Service Master**.
    3.  Notification Service Master xử lý yêu cầu:
        * Nếu theo Option A: Gọi API của từng Sub Notification Service liên quan.
        * Nếu theo Option B: Phát sự kiện lên Pub/Sub.
    4.  Các Sub Notification Service thực hiện gửi thông báo theo logic của mình.

**Tác động và Cân nhắc:**

* **Data Model:**
    * Sub Notification Service cần các bảng để lưu `notification_templates_in_tenant`, `channel_configs_in_tenant` (Zalo keys, Gmail keys, etc.), `notification_logs_in_tenant`.
    * Notification Service Master cần bảng để lưu `global_notification_templates` (nếu có), và có thể là log của các thông báo toàn cục đã gửi.
* **Cấu hình:** Mỗi Sub Notification Service sẽ có cấu hình riêng cho các kênh của mình. Superadmin App sẽ cần giao diện để cấu hình các kênh này cho từng trường.
* **API Gateway:** Cần định tuyến chính xác các request gửi thông báo đến đúng Sub Notification Service của tenant hoặc Notification Service Master.
* **User Service (Master và Sub):** Notification Service (cả Master và Sub) sẽ cần tương tác với User Service (Master hoặc Sub tương ứng) để lấy thông tin người nhận (ví dụ: email, số điện thoại liên kết Zalo, kênh ưa thích của người dùng trong tenant đó).

---
**Kết luận về Notification Service:**

Việc tách Notification Service thành cấp độ "Master" (cho các tác vụ toàn cục) và "Sub" (cho từng tenant) có vẻ là hướng đi phù hợp nhất với yêu cầu "cấu hình riêng", "dữ liệu riêng" và "Frontend App riêng" của từng trường. Nó đảm bảo sự linh hoạt, cách ly và khả năng tùy biến cao.

Mô hình "Separate Stacks per Tenant" tiếp tục được củng cố với cách tiếp cận này cho Notification Service.

Chúng ta cần bổ sung thiết kế chi tiết (SDD, Interface Contract, Data Model, OpenAPI spec) cho cả Notification Service Master và Sub Notification Service vào danh sách công việc của mình, John nhé.