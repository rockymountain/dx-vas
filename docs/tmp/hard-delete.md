## Tiêu Chí Xác Định Objects Không Nên Hard Delete trong Hệ thống dx_vas

Một object (bản ghi dữ liệu) trong hệ thống dx_vas nên được cân nhắc **không hard delete** (mà thay vào đó sử dụng soft delete hoặc cơ chế vô hiệu hóa/lưu trữ) nếu nó thỏa mãn một hoặc nhiều tiêu chí sau:

1.  **Có Ý Nghĩa Lịch Sử hoặc Cần Thiết cho Audit Trail (Tính Truy Vết):**
    * **Mô tả:** Object này chứa thông tin mà việc mất đi sẽ làm ảnh hưởng đến khả năng truy vết các hành động trong quá khứ, phân tích sự cố, hoặc hiểu được lịch sử thay đổi của hệ thống.
    * **Ví dụ trong dx_vas:**
        * `users_global`: Thông tin người dùng, ngay cả khi họ không còn hoạt động, vẫn quan trọng để biết ai đã làm gì. (`source 1`)
        * `user_tenant_assignments`: Lịch sử việc một người dùng được gán vào tenant nào và khi nào. (`source 1`)
        * `audit_logs` (từ `adr-008-audit-logging.md`): Bản chất của audit log là không được xóa. (`source 3`)
        * Các bản ghi giao dịch tài chính, điểm số, lịch sử học tập của học sinh (trong SIS, LMS).

2.  **Có Mối Quan Hệ Tham Chiếu Quan Trọng (Referential Integrity):**
    * **Mô tả:** Object này được tham chiếu bởi nhiều object khác dưới dạng khóa ngoại. Việc xóa vật lý nó có thể gây ra lỗi tham chiếu (dangling references) hoặc yêu cầu xóa hàng loạt (cascading delete) các dữ liệu liên quan mà chúng ta muốn giữ lại.
    * **Ví dụ trong dx_vas:**
        * `users_global`: Được tham chiếu bởi `user_tenant_assignments`, `audit_logs` (ví dụ: `actor_id`). (`source 1`)
        * `tenants`: Được tham chiếu bởi `user_tenant_assignments`, và có thể bởi nhiều dữ liệu cấu hình hoặc nghiệp vụ khác theo từng tenant. (`source 1`)
        * `roles` / `roles_in_tenant`: Nếu một vai trò đã được gán cho người dùng và các hành động của người dùng dưới vai trò đó đã được ghi log, việc xóa vai trò sẽ làm mất ngữ cảnh.
        * Hồ sơ học sinh (trong SIS): Được tham chiếu bởi điểm số, lịch sử lớp học, thông tin phụ huynh, v.v.

3.  **Yêu Cầu Pháp Lý hoặc Tuân Thủ (Legal & Compliance Requirements):**
    * **Mô tả:** Có các quy định pháp luật hoặc chính sách của tổ chức yêu cầu phải lưu trữ dữ liệu trong một khoảng thời gian nhất định, ngay cả khi đối tượng đó không còn "hoạt động".
    * **Ví dụ trong dx_vas:**
        * Dữ liệu học sinh, kết quả học tập (có thể cần lưu trữ theo quy định của Bộ Giáo dục).
        * Dữ liệu giao dịch tài chính (học phí).
        * Thông tin cá nhân (PII) có thể có quy định riêng về thời gian lưu trữ và cách xử lý khi người dùng yêu cầu xóa (ví dụ GDPR - "quyền được lãng quên", nhưng thường là anonymize hoặc pseudonymize chứ không phải xóa hoàn toàn khỏi mọi backup/log).

4.  **Cần Thiết cho Việc Khôi Phục hoặc Phân Tích Nghiệp Vụ (Business Analytics & Recovery):**
    * **Mô tả:** Dữ liệu từ object này, dù không còn "active", vẫn có giá trị cho việc phân tích xu hướng, thống kê, hoặc trong một số trường hợp, cần thiết để khôi phục một trạng thái nghiệp vụ nào đó.
    * **Ví dụ trong dx_vas:**
        * Thông tin các "lead" trong CRM: Dù một lead không chuyển đổi thành học sinh, dữ liệu về họ vẫn có thể hữu ích cho việc phân tích hiệu quả của các kênh tuyển sinh.
        * Lịch sử các khóa học, nội dung học tập trong LMS: Có thể cần để tham khảo hoặc tái sử dụng.

5.  **Khả năng Kích Hoạt Lại (Reactivation Potential):**
    * **Mô tả:** Có khả năng đối tượng đó sẽ cần được kích hoạt lại trong tương lai. Việc soft delete sẽ giúp quá trình này nhanh chóng và dễ dàng hơn là phải tạo mới hoàn toàn và khôi phục các liên kết.
    * **Ví dụ trong dx_vas:**
        * Một `tenant` (trường thành viên) tạm ngưng hoạt động và muốn quay lại. (`source 1`)
        * Một `users_global` (nhân viên) nghỉ việc nhưng sau đó quay lại làm việc. (`source 1`)
        * Một `user_tenant_assignment` bị `revoked` nhưng sau đó được cấp lại quyền. (`source 1`)

---

**Áp dụng các tiêu chí này vào dx_vas:**

Dựa trên các tiêu chí trên, hầu hết các thực thể dữ liệu chính trong dx_vas mà chúng ta đã thảo luận (users, tenants, user-tenant-assignments, roles, permissions, và các dữ liệu nghiệp vụ cốt lõi trong CRM/SIS/LMS) đều nên áp dụng cơ chế **soft delete/deactivation** thay vì hard delete.

**Cách triển khai Soft Delete/Deactivation:**
* Sử dụng một trường `status` (TEXT/ENUM) với các giá trị như `active`, `inactive`, `pending`, `archived`, `deleted`, `revoked`, v.v. tùy theo ngữ cảnh của từng object.
* Hoặc sử dụng một trường boolean như `is_deleted` (ít linh hoạt hơn nhưng đơn giản hơn).
* Logic ứng dụng (API, service) sẽ phải luôn kiểm tra trường trạng thái này khi truy vấn hoặc thực hiện thao tác.
* Cần có chính sách lưu trữ và dọn dẹp dữ liệu (data retention/archival/purging) rõ ràng cho các bản ghi đã được "xóa logic" sau một khoảng thời gian nhất định, nếu cần.

Việc thống nhất các tiêu chí này sẽ giúp chúng ta đưa ra quyết định nhất quán khi thiết kế Data Model cho các service còn lại và khi xem xét các yêu cầu thay đổi trong tương lai.

---

Dựa trên các tiêu chí rất rõ ràng mà bạn đã đặt ra để xác định những object không nên hard delete, và với hiểu biết của tôi về kiến trúc hệ thống dx_vas mà chúng ta đã xây dựng, đây là danh sách các object chính không nên/không được hard delete:

1.  **`users_global` (User Service Master):**
    * **Tiêu chí áp dụng:** Có ý nghĩa lịch sử/audit, có mối quan hệ tham chiếu quan trọng (đến `user_tenant_assignments`, log audit), có khả năng kích hoạt lại. (`source 1`)
    * **Giải pháp:** Sử dụng trường `status` (ví dụ: `active`, `inactive`, `deleted`) để quản lý vòng đời.

2.  **`tenants` (User Service Master):**
    * **Tiêu chí áp dụng:** Có ý nghĩa lịch sử/audit, có mối quan hệ tham chiếu quan trọng (đến `user_tenant_assignments`, cấu hình của tenant), có khả năng kích hoạt lại. (`source 1`)
    * **Giải pháp:** Sử dụng trường `status` (`active`, `suspended`, `deleted`) như đã có trong `user-service/master/data-model.md`. (`source 1`)

3.  **`user_tenant_assignments` (User Service Master):**
    * **Tiêu chí áp dụng:** Có ý nghĩa lịch sử/audit (ai được gán vào trường nào, khi nào), có mối quan hệ tham chiếu (đến `users_global` và `tenants`). (`source 1`)
    * **Giải pháp:** Sử dụng trường `assignment_status` (`active`, `revoked`) như đã có trong `user-service/master/data-model.md`. (`source 1`)

4.  **`roles_in_tenant` (Sub User Service - Cấp Tenant):**
    * **Tiêu chí áp dụng:** Có ý nghĩa lịch sử/audit (nếu vai trò này đã từng được gán và gây ra các hành động được log), có mối quan hệ tham chiếu (từ `user_role_in_tenant`).
    * **Giải pháp:** Nên có trường `status` (ví dụ: `active`, `deprecated`, `archived`) hoặc `is_active BOOLEAN`.

5.  **`global_roles_templates` (User Service Master):** (`source 1`)
    * **Tiêu chí áp dụng:** Có ý nghĩa lịch sử (template nào đã từng tồn tại), có thể được tham chiếu (ví dụ: Sub User Service đã clone và tùy chỉnh từ template này).
    * **Giải pháp:** Có thể thêm trường `status` (ví dụ: `active`, `deprecated`) hoặc `is_archived BOOLEAN`. API `DELETE` hiện tại có thể được hiểu là "archive" hoặc "deprecate" thay vì xóa vật lý.

6.  **`permissions_in_tenant` (Sub User Service - Cấp Tenant) và `global_permissions_templates` (User Service Master):** (`source 1`)
    * **Tiêu chí áp dụng:** Tương tự `roles`. Permissions thường được định nghĩa khá tĩnh. (`source 1`)
    * **Giải pháp:** Nếu có thay đổi, nên xem xét việc "deprecate" một permission code và giới thiệu một code mới thay vì xóa hẳn. Nếu quản lý động qua API (đối với template ở Master), cũng nên có trường `status` hoặc `is_archived`.

7.  **Hồ sơ Học sinh (Student Records - trong SIS và được tham chiếu bởi các Adapters):**
    * **Tiêu chí áp dụng:** Có ý nghĩa lịch sử/audit (học bạ, quá trình học), có mối quan hệ tham chiếu quan trọng (điểm số, lớp học, thông tin phụ huynh), yêu cầu pháp lý/tuân thủ (lưu trữ hồ sơ giáo dục), khả năng kích hoạt lại (học sinh quay lại học).
    * **Giải pháp:** Luôn sử dụng các trường trạng thái như `enrollment_status` (`đang học`, `đã tốt nghiệp`, `đã chuyển trường`, `đã nghỉ học`), `is_archived`.

8.  **Bản ghi Điểm số, Kết quả Học tập (trong SIS, LMS):**
    * **Tiêu chí áp dụng:** Có ý nghĩa lịch sử/audit, yêu cầu pháp lý/tuân thủ.
    * **Giải pháp:** Không bao giờ xóa. Có thể archive sau một thời gian dài.

9.  **Thông tin Học phí, Giao dịch Tài chính (trong SIS hoặc hệ thống kế toán nếu có):**
    * **Tiêu chí áp dụng:** Có ý nghĩa lịch sử/audit, yêu cầu pháp lý/tuân thủ (luật kế toán, thuế).
    * **Giải pháp:** Không bao giờ xóa. Cần chính sách lưu trữ dài hạn.

10. **`audit_logs` (Toàn hệ thống, quản lý bởi Audit Logging Service hoặc ghi nhận trong từng service):**
    * **Tiêu chí áp dụng:** Bản chất của audit log là không được xóa tùy tiện. (`source 3`)
    * **Giải pháp:** Có chính sách lưu trữ (retention policy) rõ ràng, di chuyển log cũ sang cold storage.

11. **Leads, Contacts (trong CRM):**
    * **Tiêu chí áp dụng:** Có thể cần cho phân tích nghiệp vụ, lịch sử tương tác.
    * **Giải pháp:** Nên soft delete hoặc archive sau một thời gian không hoạt động, thay vì hard delete ngay.

**Nguyên tắc chung áp dụng:**

Khi một object (bản ghi) đã được tạo ra và có khả năng đã tương tác hoặc liên kết với các phần khác của hệ thống, hoặc có giá trị lịch sử/pháp lý, thì việc xóa vật lý nó là một rủi ro. Soft delete (thay đổi trạng thái) luôn là lựa chọn an toàn và được ưu tiên. Hard delete chỉ nên được xem xét cho các dữ liệu tạm thời, dữ liệu nháp không có giá trị lâu dài, hoặc trong các quy trình dọn dẹp dữ liệu (purging) có kiểm soát chặt chẽ và tuân thủ chính sách.