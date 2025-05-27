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
Chào John,

Tuyệt vời! Bạn đã chọn **Option B: Master Phát Sự kiện lên Pub/Sub, Các Sub Notification Service Lắng Nghe** cho việc Notification Service Master điều phối gửi thông báo đến nhiều tenant. Đây là một lựa chọn tốt, hướng đến sự linh hoạt và khả năng mở rộng cho hệ thống.

Dưới đây là tóm tắt giải pháp này, bao gồm cả các phương án phòng ngừa rủi ro tiềm ẩn:

---

## Tóm tắt Giải Pháp Gửi Thông Báo Đa Tenant (Option B) và Phương Án Phòng Ngừa

**1. Mô Tả Giải Pháp (Option B):**

* **Notification Service Master nhận yêu cầu:** Khi Superadmin muốn gửi thông báo đến nhiều trường (hoặc tất cả), yêu cầu sẽ được gửi đến Notification Service Master.
* **Master Phát Sự kiện (Publish Event):**
    * Notification Service Master sẽ soạn một thông điệp sự kiện (ví dụ: `global_notification_requested`).
    * Payload của sự kiện sẽ chứa nội dung thông báo gốc và các thông tin định danh người nhận hoặc tiêu chí chọn người nhận (ví dụ: `target_tenant_ids: ["all"]` hoặc `target_tenant_ids: ["tenant_A", "tenant_B"]`, hoặc `target_user_roles: ["teacher"]` nếu Master có thông tin này).
    * Sự kiện này được publish lên một topic chung trên Google Cloud Pub/Sub (ví dụ: `vas-global-notifications-topic`).
* **Sub Notification Services Lắng Nghe (Subscribe & Process):**
    * Mỗi Sub Notification Service (của từng trường thành viên) sẽ có một subscription lắng nghe topic `vas-global-notifications-topic` này.
    * Khi nhận được sự kiện, Sub Notification Service sẽ:
        * Kiểm tra xem sự kiện có áp dụng cho tenant của mình không (dựa trên `target_tenant_ids` hoặc các tiêu chí khác trong payload).
        * Nếu có, nó sẽ lấy nội dung thông báo gốc, áp dụng các template riêng của tenant (nếu có), xác định danh sách người nhận cụ thể trong tenant của mình (có thể cần gọi Sub User Service của tenant đó).
        * Sử dụng cấu hình kênh gửi riêng (Zalo OA, Gmail API keys, etc.) của tenant để gửi thông báo.
        * Ghi log và theo dõi trạng thái gửi thông báo trong phạm vi tenant của mình.
* **Decoupling và Xử lý Bất Đồng bộ:** Master Service không cần biết chi tiết về từng Sub Service, và việc gửi thông báo diễn ra hoàn toàn bất đồng bộ.

**2. Phương Án Phòng Ngừa Rủi Ro Tiềm Ẩn:**

* **Rủi ro: Khó theo dõi trạng thái gửi tổng thể từ Master.**
    * **Phòng ngừa:**
        * **Sự kiện Phản hồi (Acknowledgement Event):** Mỗi Sub Notification Service sau khi xử lý (gửi thành công, thất bại, hoặc đưa vào hàng đợi nội bộ) sẽ phát một sự kiện phản hồi (ví dụ: `tenant_notification_batch_status`) lên một topic Pub/Sub khác (ví dụ: `vas-tenant-notification-ack-topic`).
        * Payload của sự kiện phản hồi này sẽ chứa `tenant_id`, trạng thái xử lý, số lượng gửi thành công/thất bại, và có thể là `correlation_id` từ sự kiện gốc của Master.
        * Một service giám sát riêng (hoặc chính Notification Service Master nếu được thiết kế để lắng nghe) có thể thu thập các sự kiện phản hồi này để tổng hợp báo cáo trạng thái cho Superadmin.
        * Sử dụng một cơ sở dữ liệu (ví dụ: BigQuery hoặc một bảng trong PostgreSQL của Master) để lưu trữ trạng thái chi tiết của từng yêu cầu gửi thông báo toàn cục và cập nhật dựa trên các sự kiện phản hồi.

* **Rủi ro: Đảm bảo Xử lý Message Idempotent tại Sub Services.**
    * **Phòng ngừa:**
        * Mỗi sự kiện từ Master Service phải có một `message_id` hoặc `event_id` duy nhất.
        * Mỗi Sub Notification Service phải triển khai logic kiểm tra `message_id` đã được xử lý hay chưa (ví dụ: lưu vào bảng `processed_global_notifications` trong DB của Sub Service hoặc dùng Redis) để tránh gửi lại cùng một thông báo toàn cục nhiều lần.
        * Toàn bộ logic xử lý sự kiện trong Sub Service (từ lấy thông tin người nhận đến gửi thông báo) nên được đặt trong một transaction (nếu có tương tác DB) để đảm bảo tính nhất quán.

* **Rủi ro: "Broadcast Storm" hoặc Sub Service xử lý nhầm sự kiện không dành cho mình.**
    * **Phòng ngừa:**
        * Payload của sự kiện từ Master Service **phải rất rõ ràng** về việc sự kiện này dành cho những tenant nào (`target_tenant_ids`).
        * Logic lọc sự kiện ở đầu mỗi Sub Notification Service phải chặt chẽ: chỉ xử lý nếu `tenant_id` của mình nằm trong danh sách `target_tenant_ids` hoặc nếu `target_tenant_ids` là "all" (và có cơ chế phân biệt rõ ràng).

* **Rủi ro: Quá tải tại một Sub Notification Service nếu nhận được nhiều sự kiện lớn cùng lúc.**
    * **Phòng ngừa:**
        * Cấu hình Pub/Sub subscription với `flow control` để giới hạn số lượng message mà Sub Service nhận cùng một lúc.
        * Sub Notification Service nên thiết kế để xử lý bất đồng bộ nội bộ (ví dụ: nhận message từ Pub/Sub, đưa vào một hàng đợi nội bộ của service (có thể là background task, Celery, hoặc một Pub/Sub topic riêng của tenant) để từ từ xử lý việc gửi đi).
        * Đảm bảo Sub Notification Service được triển khai trên Cloud Run với cấu hình min/max instances phù hợp để có thể auto-scale.

* **Rủi ro: Lỗi cấu hình kênh gửi tại một Tenant.**
    * **Phòng ngừa:**
        * Sub Notification Service phải có cơ chế ghi log lỗi chi tiết khi không thể gửi qua một kênh nào đó.
        * Phát sự kiện phản hồi (như đã đề cập ở trên) với trạng thái lỗi và lý do cụ thể.
        * Superadmin Webapp nên có giao diện để kiểm tra và cập nhật cấu hình kênh gửi cho từng tenant.

* **Rủi ro: Dead Letter Topic (DLT) của Pub/Sub bị đầy do lỗi logic lặp đi lặp lại ở Sub Service.**
    * **Phòng ngừa:**
        * Giám sát chặt chẽ DLT của topic `vas-global-notifications-topic`.
        * Khi có message vào DLT, cần có quy trình phân tích nguyên nhân gốc (root cause) ngay.
        * Nếu lỗi do logic ở Sub Service, cần sửa lỗi và có thể phải re-publish message từ DLT (sau khi đã sửa lỗi và đảm bảo idempotency).

---

Bằng cách triển khai Option B cùng với các biện pháp phòng ngừa này, chúng ta có thể xây dựng một hệ thống gửi thông báo đa tenant mạnh mẽ, linh hoạt, có khả năng mở rộng tốt và quản lý được các rủi ro tiềm ẩn.