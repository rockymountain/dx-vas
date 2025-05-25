# 📘 Ops Guide – dx-vas

Tài liệu này dành cho DevOps/SRE vận hành hệ thống dx-vas trên môi trường production và staging. Bao gồm giám sát, triển khai, scaling, bảo mật, kiểm tra sự cố và các quy trình vận hành thường nhật.

---

## Mục lục

1. [Tổng quan triển khai hạ tầng](#1-tổng-quan-triển-khai-hạ-tầng)
2. [Triển khai dịch vụ & môi trường](#2-triển-khai-dịch-vụ--môi-trường)
3. [Giám sát & cảnh báo](#3-giám-sát--cảnh-báo)
4. [Logging & Trace phân tán](#4-logging--trace-phân-tán)
5. [Quản lý bảo mật & credentials](#5-quản-lý-bảo-mật--credentials)
6. [Chiến lược scaling & autoscale](#6-chiến-lược-scaling--autoscale)
7. [Quy trình cập nhật & rollback](#7-quy-trình-cập-nhật--rollback)
8. [Quản lý Pub/Sub & Dead Letter](#8-quản-lý-pubsub--dead-letter)
9. [Kiểm soát chi phí & tối ưu tài nguyên](#9-kiểm-soát-chi-phí--tối-ưu-tài-nguyên)
10. [Checklist định kỳ vận hành](#10-checklist-định-kỳ-vận-hành)
11. [Ứng phó sự cố & khôi phục](#11-ứng-phó-sự-cố--khôi-phục)
12. [Backup & Disaster Recovery (DR)](#12-backup--disaster-recovery-dr)
13. [Security audit & compliance](#13-security-audit--compliance)
14. [Công cụ & dashboard hỗ trợ](#14-công-cụ--dashboard-hỗ-trợ)

---

## 1. Tổng quan triển khai hạ tầng

Hệ thống dx-vas được triển khai hoàn toàn trên hạ tầng **Google Cloud Platform (GCP)**, sử dụng các dịch vụ serverless và managed để đảm bảo khả năng mở rộng, bảo trì đơn giản và vận hành ổn định.

---

### ⚙️ Kiến trúc triển khai chính

| Thành phần | Triển khai trên | Ghi chú |
|------------|-----------------|---------|
| API Gateway | Cloud Run | Kiểm soát truy cập, kiểm tra RBAC |
| Core Services (Auth, User, Notification) | Cloud Run | Độc lập, stateless |
| Business Adapters (CRM/SIS/LMS) | Cloud Run | Kết nối hệ thống bên ngoài |
| Redis Cache | Memorystore | Lưu RBAC cache, token, session |
| PostgreSQL (Core) | Cloud SQL | CSDL chính cho RBAC, người dùng |
| MySQL (Adapters) | Cloud SQL | Tương thích với hệ thống CRM/SIS/LMS |
| Pub/Sub | GCP Pub/Sub | Event bus cho Notification & Sync |
| GCS | Google Cloud Storage | Lưu trữ ảnh, file tĩnh, JSON config |

---

### ☁️ Hạ tầng phân tầng

- **Cloud Run**: Mỗi service là một container riêng biệt, autoscale theo lưu lượng.
- **Cloud SQL**: Instance riêng cho PostgreSQL (core) và MySQL (adapters), bật PITR.
- **VPC & firewall rules**: Tất cả Cloud Run → DB hoặc Redis đều qua VPC connector.
- **IAM & Secret Manager**: Quản lý phân quyền truy cập service-level và secrets.

---

### 🗂️ Môi trường triển khai

| Môi trường | Nhánh Git | Ghi chú |
|------------|-----------|---------|
| Staging | `dev` | Tự động deploy khi merge PR |
| Production | `main` | Deploy có approval |

---

📎 Sơ đồ kiến trúc tổng quan: [System Diagrams](../architecture/system-diagrams.md#9-deployment-overview-diagram--sơ-đồ-triển-khai-tổng-quan)

📎 Chi tiết RBAC: [RBAC Deep Dive](../architecture/rbac-deep-dive.md)

---

## 2. Triển khai dịch vụ & môi trường

Hệ thống dx-vas gồm nhiều service độc lập, mỗi service là một Dockerized microservice được triển khai lên **Google Cloud Run**. Mỗi service có repository riêng biệt, CI/CD riêng, và tách biệt giữa staging và production.

---

### 📦 Mô hình multi-repo

| Service | Repo | CI/CD | Port dev |
|---------|------|-------|----------|
| API Gateway | `dx-api-gateway` | GitHub Actions → Cloud Run | 8080 |
| Auth Service | `dx-auth-service` | GitHub Actions → Cloud Run | 8001 |
| User Service | `dx-user-service` | GitHub Actions → Cloud Run | 8002 |
| Notification Service | `dx-notification-service` | GitHub Actions → Cloud Run | 8003 |
| CRM Adapter | `dx-crm-adapter` | GitHub Actions → Cloud Run | 8011 |
| SIS Adapter | `dx-sis-adapter` | GitHub Actions → Cloud Run | 8012 |
| LMS Adapter | `dx-lms-adapter` | GitHub Actions → Cloud Run | 8013 |

---

### 🧱 Quy trình triển khai điển hình

1. Developer push code lên nhánh `dev`
2. CI chạy test + lint + security scan
3. Nếu pass:
   - Tự động build image → upload Container Registry (Artifact Registry)
   - Deploy lên Cloud Run môi trường Staging
4. Với `main`:
   - Chỉ được merge khi có 2 approval
   - Yêu cầu xác nhận deploy → Production

---

### 🛠 Cấu hình deploy (ví dụ GitHub Actions)

```yaml
- name: Deploy to Cloud Run
  uses: google-github-actions/deploy-cloudrun@v1
  with:
    service: user-service
    image: gcr.io/$PROJECT_ID/user-service:$GITHUB_SHA
    region: asia-southeast1
    env_vars: ENV=production
````

---

### 📁 Cấu hình biến môi trường (Cloud Run)

* Dùng Secret Manager để lưu token nhạy cảm
* Các biến `.env` được sync thông qua deploy script
* Một số biến môi trường điển hình:

```env
ENV=production
DATABASE_URL=...
REDIS_URL=...
JWT_SECRET=...
PUBSUB_PROJECT_ID=...
```

---

📎 Chi tiết CI/CD: [Dev Guide](./dev-guide.md#8-quy-trình-test--ci-cd)

📎 Cấu trúc môi trường Cloud Run: [System Diagrams](../architecture/system-diagrams.md#9-deployment-overview-diagram--sơ-đồ-triển-khai-tổng-quan)

---

## 3. Giám sát & cảnh báo

Hệ thống dx-vas sử dụng Google Cloud Monitoring để theo dõi trạng thái dịch vụ, hiệu năng, tỉ lệ lỗi và thiết lập cảnh báo theo rule tùy chỉnh. Mỗi service được giám sát độc lập và có dashboard riêng.

---

### 📊 Các loại metric giám sát

| Nhóm | Metric | Ví dụ |
|------|--------|-------|
| Cloud Run | Request count, latency, error rate | 5xx tăng bất thường |
| Pub/Sub | Pending messages, retry count | Queue bị kẹt |
| Redis | Miss rate, latency | Cache chậm hoặc mất |
| Database | Connection usage, slow queries | PostgreSQL / MySQL |
| RBAC | Permission denied, cache invalidated | Sự kiện bảo mật |
| Notification | Success/failure rate theo kênh | Zalo/Gmail thất bại |

---

### ⚠️ Alerting rule gợi ý

| Rule | Điều kiện | Severity |
|------|-----------|----------|
| Gateway error spike | 5xx > 20% trong 5 phút | 🔥 Critical |
| Redis miss tăng cao | Miss rate > 50% trong 10 phút | ⚠️ Warning |
| Pub/Sub backlog | > 500 messages trong 5 phút | ⚠️ Warning |
| Đăng nhập thất bại | > 10 lần từ 1 IP trong 1 phút | 🚨 Security |
| DB connection full | Sử dụng > 90% connection pool | 🔥 Critical |

---

### 📉 Dashboard đề xuất

- Một dashboard chung toàn hệ thống (multi-service)
- Dashboard riêng cho từng service:
  - `gateway-dashboard`
  - `auth-dashboard`
  - `user-dashboard`
  - `notification-dashboard`
- Có thể dùng dashboard template từ Google Cloud Monitoring

---

### 🔄 Cảnh báo & thông báo

- Cảnh báo gửi qua:
  - Google Chat (devops channel)
  - Email (alerts@truongvietanh.edu.vn)
  - Định kỳ audit cảnh báo sai lệch hoặc trùng lặp

---

📎 Xem thêm cấu trúc trace/log tại: [Mục 4 – Logging & Trace](#4-logging--trace-phân-tán)  

📎 Định nghĩa dashboard: [`docs/observability/`](../observability/)

---

## 4. Logging & Trace phân tán

Mọi service trong dx-vas đều ghi log theo định dạng JSON chuẩn hóa, có gắn `request_id`, và gửi về Google Cloud Logging. Hệ thống đồng thời hỗ trợ tracing phân tán với Google Cloud Trace để theo dõi luồng xử lý xuyên suốt nhiều service.

---

### 🧾 Định dạng log chuẩn

```json
{
  "timestamp": "2024-06-01T10:15:00Z",
  "level": "INFO",
  "service": "user-service",
  "request_id": "req-123456",
  "message": "User updated successfully",
  "user_id": "u789"
}
````

#### 📌 Lưu ý:

* Log theo cấu trúc (structured logging) để dễ filter và alert
* Gắn `request_id` xuyên suốt: từ client → gateway → service → pub/sub

---

### 🔍 Trace phân tán

* Mỗi request đều gắn `X-Request-ID` và `X-Trace-ID`
* Gateway khởi tạo trace → propagate xuống các service
* Google Cloud Trace ghi nhận timeline xuyên service (gọi API, DB, Pub/Sub...)

---

### 📊 Sử dụng log & trace

* Tìm theo `request_id`: truy vết lỗi logic liên service
* So sánh `start_time` – `end_time`: đo độ trễ từng hop
* Lọc theo `level: ERROR`: phát hiện service mất ổn định
* Trace các retry / fail event từ Notification, RBAC, Auth

---

### 🗂️ Gợi ý log prefix

| Thành phần   | Prefix gợi ý |
| ------------ | ------------ |
| Gateway      | `[GATEWAY]`  |
| Auth Service | `[AUTH]`     |
| User Service | `[USER]`     |
| Notification | `[NOTI]`     |
| RBAC         | `[RBAC]`     |
| Redis        | `[CACHE]`    |

---

### 🔐 Log bảo mật

* Log các sự kiện nhạy cảm:

  * Đăng nhập sai, lock tài khoản
  * Thay đổi role/permission
  * Lỗi RBAC condition

---

📎 Hướng dẫn chi tiết log/tracing: [`adr-008-audit-logging.md`](../ADR/adr-008-audit-logging.md)

📎 Cấu trúc trace → diagram: [System Diagrams](../architecture/system-diagrams.md#4-rbac-evaluation-flow--luồng-đánh-giá-phân-quyền-động)

---

## 5. Quản lý bảo mật & credentials

Tất cả thông tin nhạy cảm như token, secret, key API đều được quản lý qua **Google Secret Manager**, có kiểm soát quyền truy cập bằng IAM. Không để bất kỳ credential nào trong repo code hoặc biến môi trường công khai.

---

### 🔐 Loại thông tin cần bảo vệ

| Loại | Ví dụ |
|------|-------|
| JWT secret | `JWT_SECRET`, `REFRESH_SECRET` |
| OAuth credentials | Google OAuth Client ID/Secret |
| Redis password | `REDIS_AUTH` nếu bật ACL |
| DB credentials | `DATABASE_URL` |
| API key bên thứ ba | Zalo OA, Gmail API |
| Webhook URL | Google Chat, Slack alerting |

---

### 🧱 Quy trình quản lý secret

1. Tạo secret trên GCP:

```bash
gcloud secrets create dx-auth-jwt-secret \
  --replication-policy automatic
````

2. Upload version:

```bash
echo "supersecret123" | gcloud secrets versions add dx-auth-jwt-secret --data-file=-
```

3. Gán quyền IAM cho Cloud Run:

```bash
gcloud secrets add-iam-policy-binding dx-auth-jwt-secret \
  --member=serviceAccount:auth-service@PROJECT_ID.iam.gserviceaccount.com \
  --role=roles/secretmanager.secretAccessor
```

4. Truy xuất trong runtime (qua ENV hoặc SDK)

---

### 📦 Thực hành tốt

* Secret chỉ cấp quyền `read`, không `write` từ service
* Rotate định kỳ mỗi 90 ngày
* Bắt buộc audit nếu secret bị thay đổi
* Không để `.env` chứa secret trong repo – dùng `.env.example`

---

### 🧪 Kiểm tra nhanh

```bash
gcloud secrets list
gcloud secrets versions list dx-auth-jwt-secret
```

---

📎 Chính sách bảo mật chi tiết: [`adr-003-secrets.md`](../ADR/adr-003-secrets.md)

📎 Tổng quan bảo mật toàn hệ thống: [Dev Guide – Mục 11.1](./dev-guide.md#11-theo-dõi--vận-hành)

---

## 6. Chiến lược scaling & autoscale

Hệ thống dx-vas sử dụng **Google Cloud Run** để triển khai toàn bộ service backend. Cloud Run cung cấp khả năng autoscale theo số lượng request, đảm bảo hệ thống luôn phản hồi nhanh nhưng tiết kiệm tài nguyên.

---

### ⚙️ Autoscaling mặc định

| Service | Min instance | Max instance | Concurrency | Ghi chú |
|---------|--------------|--------------|-------------|---------|
| Gateway | 2 | 50 | 10 | Luôn duy trì sẵn sàng |
| Auth/User | 1 | 20 | 10 | Core services, nhạy cảm với độ trễ |
| Notification | 0 | 30 | 20 | Có thể burst cao khi gửi hàng loạt |
| Adapters | 0 | 10 | 10 | Tùy theo mức độ tích hợp (CRM, SIS, LMS) |

---

### 🔁 Hành vi scaling

- Tăng instance khi số lượng request vượt concurrency threshold
- Giảm instance khi idle trong một thời gian (15 phút mặc định)
- Cloud Run đảm bảo mỗi container xử lý tối đa `CONCURRENCY` request cùng lúc

---

### 📉 Tối ưu scaling

- Sử dụng health check `/healthz` để loại bỏ container lỗi khỏi autoscaler
- Log metric `request count`, `concurrency`, `latency` để đánh giá scaling thực tế
- Có thể gán hạn mức chi phí/quotas cho từng service nếu cần kiểm soát tải

---

### 🔐 Lưu ý

- Luôn bật min instance = 1 với Auth và User Service để giảm cold start
- Sử dụng VPC Connector cho Cloud SQL/Redis – đảm bảo scaling không bị nghẽn
- Không scale vô hạn: max instance cần được cấu hình phù hợp để tránh quá tải DB

---

📎 Cấu trúc triển khai: [System Diagrams](../architecture/system-diagrams.md#9-deployment-overview-diagram--sơ-đồ-triển-khai-tổng-quan)  

📎 Tham khảo thêm: [Dev Guide](./dev-guide.md#8-quy-trình-test--ci-cd)

---

## 7. Quy trình cập nhật & rollback

Hệ thống dx-vas sử dụng CI/CD để triển khai dịch vụ lên Cloud Run. Mỗi lần cập nhật đều tạo ra một bản phát hành mới (revision) và có thể rollback về bản cũ nhanh chóng nếu phát sinh sự cố.

---

### 🚀 Quy trình cập nhật

1. Developer merge code vào nhánh `dev` hoặc `main`
2. GitHub Actions thực hiện:
   - Chạy test
   - Build Docker image → đẩy lên Artifact Registry
   - Deploy Cloud Run service với revision mới
3. Cloud Run giữ lại toàn bộ revision đã deploy (trừ khi bị xóa)

---

### 📦 Kiểm soát theo môi trường

| Nhánh | Môi trường | Auto Deploy | Require Approval |
|-------|------------|-------------|------------------|
| `dev` | Staging | ✅ | ❌ |
| `main` | Production | ⚠️ (Có thể bật/tắt) | ✅ Required |

---

### 🛑 Rollback khi lỗi

Nếu sau deploy xảy ra lỗi:

1. Vào Google Cloud Console → Cloud Run → Service
2. Chọn tab **Revisions**
3. Click **Pin traffic** hoặc **Rollback** về revision trước đó
4. Hoặc dùng lệnh CLI:

```bash
gcloud run services update-traffic user-service \
  --to-revisions=rev-1234=100
````

---

### 🧪 Lưu ý khi kiểm tra version

* Mỗi revision có label: `git_sha`, `build_time`
* Gateway hoặc các service có thể trả `X-Revision-ID` trong header
* Có thể tạo endpoint `/__version__` trả về git commit + thời gian build

---

### ⚠️ Những rủi ro cần chú ý

* Rollback logic có thể không rollback được DB schema → dùng `alembic downgrade` nếu cần
* Cache hoặc RBAC có thể cần invalidate nếu rollback auth/user
* Nếu sử dụng secret mới trong phiên bản mới → rollback phải đảm bảo backward-compatible

---

📎 Hướng dẫn CI/CD chi tiết: [Dev Guide](./dev-guide.md#8-quy-trình-test--ci-cd)

📎 Checklist sự cố: [Mục 11 – Ứng phó sự cố](#11-ứng-phó-sự-cố--khôi-phục)

---

## 8. Quản lý Pub/Sub & Dead Letter

Hệ thống dx-vas sử dụng Google Cloud Pub/Sub để truyền sự kiện bất đồng bộ giữa các service (Notification, Sync, Audit...). Mỗi topic có ít nhất một subscription. Nếu xử lý lỗi hoặc timeout, message sẽ được chuyển sang Dead Letter Topic (DLT).

---

### 🔁 Các topic chính

| Topic | Publisher | Subscriber | Mô tả |
|-------|-----------|------------|-------|
| `notification-events` | CRM, SIS, Gateway | Notification Service | Gửi thông báo theo sự kiện |
| `user-events` | Auth, CRM | User Service | Đồng bộ trạng thái, RBAC |
| `sync-events` | CRM Adapter | SIS Adapter, LMS Adapter | Đồng bộ học sinh sau tuyển sinh |
| `audit-events` | All | Audit Logger | Ghi nhận hành vi nhạy cảm |

---

### 📦 Dead Letter Topics (DLT)

Mỗi subscription đều có cấu hình DLT đi kèm, ví dụ:

```bash
gcloud pubsub subscriptions update my-subscription \
  --dead-letter-topic=projects/$PROJECT_ID/topics/my-sub-dlt \
  --max-delivery-attempts=5
````

Khi message retry quá 5 lần → chuyển sang DLT → log lại sự kiện lỗi.

---

### 🔍 Giám sát Pub/Sub

* Sử dụng metric:

  * `num_undelivered_messages`
  * `ack_latency`
  * `dead_letter_message_count`
* Có thể tạo alert khi:

  * DLT tăng đột biến
  * Tồn đọng nhiều message chưa xử lý
* Dashboard: `pubsub-dashboard`, `notification-dlt-dashboard`

---

### 🛠 Handling retry

* Logic subscriber cần **idempotent** (xử lý lại không gây lỗi)
* Dùng `message_id` để detect duplicate
* Nếu sử dụng database → dùng transaction + retry-safe query

---

### ✏️ Debug DLT message

1. Subscribe tạm thời vào DLT
2. Ghi log nội dung message để phân tích
3. Quyết định: sửa lỗi source, hoặc re-publish message (thận trọng)

---

📎 Ví dụ triển khai subscriber: [`dx-notification-service`](https://github.com/vas-org/dx-notification-service/tree/main/app/events)

📎 Sơ đồ flow: [System Diagrams](../architecture/system-diagrams.md#5-data-synchronization-flow--đồng-bộ-học-sinh-crm--sis--lms)

---

## 9. Kiểm soát chi phí & tối ưu tài nguyên

Hệ thống dx-vas sử dụng nhiều dịch vụ managed (Cloud Run, SQL, Pub/Sub...) – giúp vận hành đơn giản nhưng cũng cần theo dõi sát để tránh chi phí vượt kiểm soát. Chi phí chủ yếu phát sinh từ Cloud Run, Cloud SQL, Pub/Sub, Storage và Network.

---

### 💰 Các thành phần tốn chi phí

| Dịch vụ | Nhóm chính | Chi phí tính theo |
|---------|------------|-------------------|
| Cloud Run | API Gateway, các service | Số request, CPU/Memory usage |
| Cloud SQL | PostgreSQL, MySQL | Instance uptime + storage |
| Redis (MemoryStore) | RBAC cache | GB RAM provisioned |
| Pub/Sub | Event-driven flow | Message volume + retention |
| GCS | Static assets, backup | GB lưu trữ + truy xuất |
| IAM/Secret Manager | Token, config | Rất nhỏ nhưng có định mức |

---

### 📊 Theo dõi chi phí

- Sử dụng Cloud Billing → Cost Table + Cost Breakdown theo service
- Tạo **budget alert** qua Billing Budgets:

```bash
gcloud billing budgets create --display-name="VAS Monthly Budget" ...
````

* Gửi cảnh báo qua email hoặc Google Chat

---

### 🎯 Chiến lược tối ưu

| Dịch vụ   | Chiến lược                                                      |
| --------- | --------------------------------------------------------------- |
| Cloud Run | Set max instance limit, tuning concurrency                      |
| Redis     | Chỉ cache permission nóng, TTL ngắn (5–10 phút)                 |
| Cloud SQL | Tắt instance staging ban đêm, sử dụng shared tier nếu load thấp |
| Pub/Sub   | Giảm message size, kiểm tra retry quá mức                       |
| GCS       | Xoá ảnh cũ/backup quá hạn, dùng `lifecycle rule`                |

---

### ⚠️ Cảnh báo thực tế

* Cloud SQL chiếm \~50% tổng bill nếu không tối ưu idle time
* Redis billing theo GB RAM provisioned, không autoscale
* Pub/Sub có thể phát sinh retry loop gây bill ẩn

---

📎 Theo dõi chi phí chi tiết: GCP → Billing → Reports

📎 Cảnh báo sử dụng resource: Xem thêm [Mục 3 – Giám sát & cảnh báo](#3-giám-sát--cảnh-báo)

---

## 10. Checklist định kỳ vận hành

Việc kiểm tra định kỳ là thiết yếu để đảm bảo hệ thống dx-vas hoạt động an toàn, hiệu quả và không phát sinh chi phí ngoài kiểm soát. Checklist sau đây được chia theo tần suất: hàng tuần, hàng tháng và hàng quý.

---

### 🔁 Hàng tuần

- [ ] Kiểm tra Cloud Run error rate ≥ 5%
- [ ] Xác minh các alert Pub/Sub/Redis có bị miss không
- [ ] Xem lại DLT logs → xử lý message fail lặp lại
- [ ] Kiểm tra trạng thái deploy của các service staging
- [ ] Check audit log liên quan đến RBAC, Auth

---

### 📅 Hàng tháng

- [ ] Xoá Cloud SQL backups cũ (nếu không cần thiết)
- [ ] Review budget report → so với usage thực tế
- [ ] Rotate JWT signing key (nếu bật chế độ ngắn hạn)
- [ ] Check Cloud Storage → xoá ảnh/file/log quá hạn
- [ ] Chạy lại test E2E: Webform → CRM → SIS → LMS
- [ ] Kiểm tra lại giới hạn Cloud Run (max instance, timeout)

---

### 📣 Hàng quý

- [ ] Rà soát IAM roles trên toàn bộ dự án
- [ ] Đánh giá lại quota của Redis, Cloud SQL, Pub/Sub
- [ ] Kiểm thử kịch bản rollback và DR (Disaster Recovery)
- [ ] Đánh giá performance thực tế → điều chỉnh autoscale
- [ ] Cập nhật bảng permission & roles (RBAC evolution)

---

### 📌 Cách theo dõi checklist

- Lưu checklist trên Notion/Google Sheet theo ngày
- Dùng Google Calendar để nhắc lịch định kỳ
- Gắn vào buổi họp kỹ thuật hàng tháng (KT Sync)

---

📎 Các log hỗ trợ: [Mục 4 – Logging & Trace](#4-logging--trace-phân-tán)  

📎 Chi phí/quota → [Mục 9 – Kiểm soát chi phí](#9-kiểm-soát-chi-phí--tối-ưu-tài-nguyên)

---

## 11. Ứng phó sự cố & khôi phục

Trong mọi hệ thống sản phẩm, khả năng phản ứng nhanh khi xảy ra lỗi nghiêm trọng là yếu tố sống còn. dx-vas áp dụng mô hình chuẩn hóa: phân loại sự cố, xử lý theo checklist, ghi nhận hậu kiểm (postmortem).

---

### 🚨 Phân loại sự cố

| Loại | Ví dụ | Phản ứng |
|------|-------|----------|
| 🔥 Production Outage | Cloud Run 5xx toàn hệ thống | Phát động kênh khẩn cấp |
| ⚠️ Functional Bug | Notification không gửi | Theo mức độ nghiêm trọng |
| ⏱ Degraded | Redis timeout, DB latency cao | Theo dõi, điều chỉnh autoscale |
| 🔒 Security | Rò rỉ token, tấn công brute force | Ngắt tạm, rotate credential |

---

### 🧯 Checklist phản ứng nhanh

1. **Xác nhận lỗi**
   - Ghi nhận timestamp + request_id
   - Check status Cloud Run, Redis, Pub/Sub, GCS
2. **Phân loại lỗi**
   - Outage / Degraded / Security
3. **Kích hoạt recovery**
   - Nếu lỗi rollout: Rollback ngay (Xem [Mục 7](#7-quy-trình-cập-nhật--rollback))
   - Nếu Redis fail: flush cache + restart service
4. **Thông báo**
   - Gửi update qua Google Chat `#dx-ops-alert`
5. **Khôi phục tạm thời**
   - Tắt 1 vài feature non-critical nếu cần

---

### 🗂 Log & công cụ hỗ trợ

- Cloud Run log + trace
- Pub/Sub Dead Letter inspection
- Redis monitor command
- GCS object viewer (xem cấu hình/static lỗi)

---

### 📋 Postmortem template

1. **Sự cố:** (mô tả ngắn)
2. **Thời gian:** (bắt đầu – kết thúc)
3. **Ảnh hưởng:** (người dùng nào? dịch vụ nào?)
4. **Nguyên nhân gốc:** (root cause)
5. **Giải pháp khắc phục:** (ngắn hạn + dài hạn)
6. **Ai xử lý / liên hệ:** (contact)

> ✅ Mọi postmortem cần lưu tại `docs/postmortem/` dưới repo `dx-vas`

---

📎 Cách kiểm tra sự cố theo trace: [Mục 4 – Logging & Trace](#4-logging--trace-phân-tán)  

📎 Khả năng rollback: [Mục 7 – Cập nhật & Rollback](#7-quy-trình-cập-nhật--rollback)

---

## 12. Backup & Disaster Recovery (DR)

Hệ thống dx-vas được thiết kế với khả năng khôi phục linh hoạt và giới hạn downtime. Mỗi thành phần quan trọng đều có chiến lược DR riêng, đảm bảo khôi phục dữ liệu và dịch vụ khi xảy ra sự cố lớn (mất DB, hạ tầng, lỗi logic nghiêm trọng...).

---

### 🎯 RTO & RPO mục tiêu

| Thành phần | RTO | RPO |
|------------|-----|-----|
| PostgreSQL (Core) | 15 phút | 5 phút |
| Redis (Cache) | 5 phút | 0 (chấp nhận mất cache) |
| MySQL (Adapters) | 30 phút | 15 phút |
| Pub/Sub | 10 phút | 1 phút |
| Static (GCS) | 30 phút | 0 phút |

> RTO: Recovery Time Objective  
> RPO: Recovery Point Objective

---

### 📦 Backup chiến lược

| Thành phần | Phương pháp |
|------------|-------------|
| Cloud SQL (PostgreSQL, MySQL) | Enable PITR + Scheduled Snapshots |
| Redis | Không backup – auto recreate từ source |
| GCS | Object versioning + lifecycle rule |
| Pub/Sub DLT | Lưu message fail trong Dead Letter Topic |
| Interface Contracts & ADRs | Sync từ Git → giữ bản chính trên `dx-vas` repo |

---

### 🔁 Kiểm thử DR định kỳ

- Mỗi quý thực hiện drill (giả lập sự cố)
  - Xóa bản ghi DB staging → restore từ snapshot
  - Simulate fail của Notification hoặc Gateway
- Đảm bảo khôi phục service trong khung thời gian mục tiêu

---

### 🔐 Lưu ý bảo mật

- Backup GCS phải bật encryption at rest
- Chỉ tài khoản có `roles/cloudsql.admin` mới được tạo snapshot
- Audit access các bản backup qua Cloud Audit Log

---

### 🧪 Công cụ kiểm tra & khôi phục

```bash
# Liệt kê backup
gcloud sql backups list --instance=dx-user-postgres

# Restore từ backup
gcloud sql instances restore-backup dx-user-postgres \
  --backup-id=1234567890
````

---

📎 Kịch bản sự cố toàn cụm: [Mục 11 – Ứng phó sự cố & khôi phục](#11-ứng-phó-sự-cố--khôi-phục)

📎 Cấu trúc triển khai dịch vụ: [System Diagrams](../architecture/system-diagrams.md#9-deployment-overview-diagram--sơ-đồ-triển-khai-tổng-quan)

---

## 13. Security audit & compliance

Hệ thống dx-vas áp dụng cơ chế kiểm soát bảo mật chặt chẽ ngay từ kiến trúc. Đội DevOps/SRE cần duy trì các hoạt động audit định kỳ để phát hiện lỗ hổng, kiểm soát phân quyền và đảm bảo tuân thủ theo các tiêu chuẩn bảo mật nội bộ.

---

### 🔍 Kiểm tra định kỳ

| Nhóm | Hành động |
|------|-----------|
| IAM | Rà soát service account, roles không dùng |
| Secret | Xác minh secret hết hạn, rotate đúng lịch |
| RBAC | So khớp permission → role → user |
| Log | Kiểm tra trace bất thường, login fail |
| Infra | Check port mở, endpoint public (Cloud Run, Redis) |

---

### 📅 Lịch audit gợi ý

| Tần suất | Hành động |
|----------|-----------|
| Hàng tháng | Rà IAM role, xóa service account cũ |
| Hàng quý | Rotate JWT secret, kiểm tra RBAC file |
| 6 tháng | Penetration test nội bộ (black box / white box) |
| Hàng năm | Audit toàn hệ thống, backup policy, access policy |

---

### 🛠 Công cụ hỗ trợ audit

- Google Cloud Asset Inventory (IAM, Secret, API enabled)
- GCP IAM Recommender
- `gcloud iam roles list`, `gcloud secrets versions list`
- Manual RBAC linter với file `permissions.yaml`

---

### 🔒 Compliance (nội bộ)

| Yêu cầu | Tình trạng |
|--------|------------|
| RBAC được kiểm soát qua giao diện & versioned | ✅ |
| Không có secret nằm trong repo code | ✅ |
| Tất cả dịch vụ qua Gateway, có log RBAC | ✅ |
| Xoá user = inactive + revoke token | ✅ |
| Có log & alert cho hành vi bất thường | ✅ |

> 🎯 dx-vas hướng đến mô hình **Security-by-Design** – không xử lý bảo mật sau cùng, mà tích hợp xuyên suốt.

---

📎 Chính sách RBAC: [RBAC Deep Dive](../architecture/rbac-deep-dive.md)  

📎 Chi tiết bảo mật triển khai: [`adr-004-security.md`](../ADR/adr-004-security.md)  

📎 Hệ thống log giám sát: [Mục 4 – Logging & Trace](#4-logging--trace-phân-tán)

---

## 14. Công cụ & dashboard hỗ trợ

Để hỗ trợ vận hành hệ thống dx-vas ổn định và chủ động, đội DevOps cần sử dụng kết hợp nhiều công cụ giám sát, quan sát, phân tích log và kiểm tra hạ tầng.

---

### 🧰 Bộ công cụ chính

| Công cụ | Mục đích | Ghi chú |
|--------|----------|---------|
| Google Cloud Logging | Xem log toàn hệ thống | Filter theo request_id, service |
| Cloud Monitoring | Theo dõi metric, alert | Định ngưỡng cho Redis, DB, Pub/Sub |
| Cloud Trace | Xem trace phân tán | Tìm request chậm, lỗi chuỗi |
| Cloud SQL Insights | Phân tích query chậm | Hỗ trợ cả Postgres & MySQL |
| Pub/Sub Viewer | Xem backlog, message retry | Kiểm tra DLT |
| Secret Manager | Quản lý key/token | Audit access |
| IAM Recommender | Đề xuất giảm quyền IAM | Audit vai trò không cần thiết |
| Artifact Registry | Quản lý Docker image | Theo dõi size, version deploy |

---

### 📊 Dashboard tham khảo

| Tên dashboard | Mục tiêu |
|----------------|-----------|
| `gateway-dashboard` | Theo dõi tỉ lệ lỗi, độ trễ Gateway |
| `auth-dashboard` | Login rate, OTP fail, token issue |
| `user-dashboard` | Truy xuất RBAC, update user |
| `notification-dashboard` | Gửi email/Zalo, theo dõi retry |
| `pubsub-dashboard` | Backlog, fail, DLT monitor |
| `db-performance-dashboard` | Slow query, connection pool usage |
| `cost-dashboard` | Tổng chi phí, cảnh báo vượt budget |

---

### 📎 Đường dẫn hữu ích

- [Cloud Run logs](https://console.cloud.google.com/run)
- [Pub/Sub Viewer](https://console.cloud.google.com/cloudpubsub/topic/list)
- [SQL Insights](https://console.cloud.google.com/sql/insights)
- [Secret Manager](https://console.cloud.google.com/security/secret-manager)

---

📎 Mọi dashboard nên được lưu trữ version (JSON export) tại:  
`dx-vas/docs/observability/dashboards/`

📎 Liên kết CI/CD – xem Dev Guide: [Dev Guide](./dev-guide.md#8-quy-trình-test--ci-cd)

