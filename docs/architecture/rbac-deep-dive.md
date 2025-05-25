# Kiến trúc Đăng nhập & Phân quyền động (RBAC) – Hệ thống dx-vas

Tài liệu này trình bày chi tiết cách hệ thống dx-vas thực hiện việc xác thực và phân quyền người dùng một cách động (RBAC – Role-Based Access Control) thông qua các thành phần chính như API Gateway, User Service, Auth Service và Redis Cache.

## 📚 Mục lục

1. [Triết lý thiết kế](#1-triết-lý-thiết-kế)
2. [Thành phần tham gia](#2-thành-phần-tham-gia)
3. [Luồng xác thực & phân quyền](#3-luồng-xác-thực--phân-quyền)
4. [Cấu trúc dữ liệu RBAC](#4-cấu-trúc-dữ-liệu-rbac)
5. [Permission có điều kiện (`condition` JSONB)](#5-permission-có-điều-kiện-condition-jsonb)
6. [Chiến lược cache RBAC & vô hiệu hóa](#6-chiến-lược-cache-rbac--vô-hiệu-hóa)
7. [Hiệu năng & khả năng mở rộng](#7-hiệu-năng--khả-năng-mở-rộng)
8. [Bảo mật chuyên sâu cho RBAC](#8-bảo-mật-chuyên-sâu-cho-rbac)
9. [Giám sát & gỡ lỗi RBAC](#9-giám-sát--gỡ-lỗi-rbac)
10. [Best Practices trong quản trị role/permission](#10-best-practices-trong-quản-trị-rolepermission)
11. [Công cụ quản trị](#11-công-cụ-quản-trị)
12. [Tài liệu liên quan](#12-tài-liệu-liên-quan)

---

## 1. Triết lý thiết kế

Hệ thống dx-vas được thiết kế dựa trên triết lý phân quyền động, đảm bảo mỗi hành động của người dùng trong hệ thống đều được kiểm soát chặt chẽ, linh hoạt và có thể mở rộng theo bối cảnh thực tế của ngành giáo dục.

Các nguyên tắc cốt lõi:

- **RBAC động, dựa trên context (contextual RBAC):** không chỉ dựa vào vai trò tĩnh mà còn đánh giá điều kiện cụ thể trong mỗi request.
- **Condition-Based Access Control:** mọi quyền đều có thể đi kèm với điều kiện được biểu diễn dưới dạng JSONB.
- **Phân tách rõ vai trò và quyền hạn:** vai trò định danh nhiệm vụ, còn permission định nghĩa rõ hành động cụ thể và phạm vi được phép.
- **Không embed permission trong JWT:** permissions được tra cứu realtime từ Redis để đảm bảo tính nhất quán, linh hoạt, và hỗ trợ cập nhật động.
- **Chống đặc quyền vượt mức:** người dùng chỉ được gán role phù hợp, quyền được kiểm soát cấp phát, audit đầy đủ.


---

## 2. Thành phần tham gia

| Thành phần        | Vai trò                                                                 |
|-------------------|--------------------------------------------------------------------------|
| **Người dùng**     | Học sinh, phụ huynh, giáo viên, nhân viên – có thể đăng nhập qua OAuth2/OTP |
| **Frontend App**   | Gửi request kèm JWT tới API Gateway sau khi đăng nhập thành công       |
| **Auth Service**   | Xác thực OAuth2 hoặc OTP → phát hành JWT (`access_token`, `refresh_token`) |
| **API Gateway**    | Điểm kiểm soát trung tâm: xác thực JWT, tra quyền từ Redis, evaluate condition |
| **User Service**   | Cung cấp role, permission và `condition` theo `user_id` qua API/Redis; phát sự kiện khi thay đổi quyền |
| **Redis Cache**    | Lưu danh sách permission đã evaluate – key theo user_id để tăng tốc truy xuất |
| **Backend Services** | Nhận request đã qua kiểm duyệt; chỉ kiểm tra `X-Permissions` header – không cần decode JWT |
| **Audit Logging**  | Lưu toàn bộ hành vi phân quyền, bao gồm gán quyền, kiểm tra RBAC, thay đổi vai trò |

> Tất cả thành phần này được kết nối qua API Gateway – là trung tâm đánh giá bảo mật RBAC động của toàn hệ thống.

---

## 3. Luồng xác thực & phân quyền

Luồng xử lý một request từ người dùng sẽ đi qua các bước:

1. Người dùng đăng nhập qua OAuth2 (GV/NV/HS) hoặc OTP (Phụ huynh)
2. Nhận `access_token` (JWT) từ Auth Service
3. Gửi request tới API Gateway (kèm JWT trong header `Authorization`)
4. API Gateway thực hiện:
   - Xác thực token (decode + verify)
   - Kiểm tra trạng thái `is_active` của user
   - Truy vấn Redis (hoặc fallback DB) để lấy `role`, `permissions`, `condition`
   - Evaluate `condition` theo context hiện tại (VD: student_id trong URL)
   - Nếu hợp lệ → forward request đến backend kèm:
     - `X-User-ID`, `X-Role`, `X-Auth-Method`, `X-Permissions`, `Trace-ID`

5. Backend Service chỉ cần kiểm tra `X-Permissions` để xác nhận quyền.

### 🔄 Sequence Diagram (mô tả logic tổng quát)

```mermaid
sequenceDiagram
    participant User
    participant Frontend
    participant AuthService
    participant APIGateway
    participant Redis
    participant UserService
    participant Backend

    User->>Frontend: Đăng nhập (OAuth2 / OTP)
    Frontend->>AuthService: Yêu cầu xác thực
    AuthService-->>Frontend: access_token (JWT)
    Frontend->>APIGateway: Gọi API (Authorization: Bearer JWT)

    APIGateway->>AuthService: Xác thực JWT (nội bộ)
    APIGateway->>UserService: Kiểm tra is_active
    APIGateway->>Redis: Lấy role + permission
    alt Cache miss
        APIGateway->>UserService: Truy vấn role/permission từ DB
    end
    APIGateway->>APIGateway: Evaluate condition
    APIGateway->>Backend: Forward request + headers

    Backend->>APIGateway: Xử lý business logic
````

> Lưu ý: Để đơn giản hóa vận hành và tăng hiệu năng, Backend không cần decode JWT hay kiểm tra RBAC lại.

📎 Xem sơ đồ minh họa luồng RBAC tại: 👉 [RBAC Evaluation Flow – System Diagrams](./system-diagrams.md#4-rbac-evaluation-flow--luồng-đánh-giá-phân-quyền-động)

## 4. Cấu trúc dữ liệu RBAC

### Bảng `users`

```sql
CREATE TABLE users (
  id UUID PRIMARY KEY,
  email TEXT UNIQUE NOT NULL,
  name TEXT NOT NULL,
  auth_provider TEXT NOT NULL DEFAULT 'google',
  user_category TEXT NOT NULL CHECK (user_category IN ('student', 'teacher', 'staff', 'parent')),
  password_hash TEXT,
  is_active BOOLEAN NOT NULL DEFAULT TRUE,
  created_at TIMESTAMP DEFAULT now(),
  updated_at TIMESTAMP DEFAULT now()
);
````

### Bảng `roles`

```sql
CREATE TABLE roles (
  id UUID PRIMARY KEY,
  name TEXT UNIQUE NOT NULL,
  description TEXT
);
```

### Bảng `permissions`

```sql
CREATE TABLE permissions (
  id UUID PRIMARY KEY,
  code TEXT UNIQUE NOT NULL,
  resource TEXT NOT NULL,
  action TEXT NOT NULL,
  condition JSONB -- e.g. { \"student_ids\": [\"abc123\"], \"campus\": \"HCM\" }
);
```

### Bảng ánh xạ

```sql
-- Một user có thể có nhiều role
CREATE TABLE user_role (
  user_id UUID REFERENCES users(id),
  role_id UUID REFERENCES roles(id),
  PRIMARY KEY (user_id, role_id)
);

-- Một role có thể có nhiều permission
CREATE TABLE role_permission (
  role_id UUID REFERENCES roles(id),
  permission_id UUID REFERENCES permissions(id),
  PRIMARY KEY (role_id, permission_id)
);
```

> Các bảng trên sẽ được migrate tĩnh và đồng bộ hóa sang cache Redis để truy xuất nhanh tại Gateway.

---

## 5. Permission có điều kiện (`condition` JSONB)

Mỗi permission có thể đi kèm một điều kiện (`condition`) dưới dạng JSONB, cho phép hệ thống xác định ngữ cảnh mà quyền đó có hiệu lực. Điều này mang lại sự linh hoạt trong phân quyền theo từng học sinh, lớp học, campus, hoặc trạng thái học tập cụ thể.

### 🔍 Ví dụ đơn giản

```json
{
  "code": "VIEW_SCORE_OWN_CHILD",
  "resource": "student_score",
  "action": "view",
  "condition": {
    "student_ids": ["stu-123"]
  }
}
````

> Quyền này chỉ áp dụng nếu học sinh đang được truy cập nằm trong danh sách `student_ids` mà người dùng được phép xem.

---

### 🔁 Ví dụ nâng cao: điều kiện AND

```json
{
  "code": "EDIT_SCORE_CLASS_OWNER",
  "resource": "student_score",
  "action": "edit",
  "condition": {
    "class_id": "cls-10a",
    "subject_id": "math"
  }
}
```

> Chỉ giáo viên có cả `class_id = cls-10a` và `subject_id = math` mới được chỉnh sửa điểm của học sinh trong lớp.

---

### 🚫 Ví dụ điều kiện loại trừ

```json
{
  "code": "VIEW_SCORE_EXCLUDE_SPECIAL_PROGRAM",
  "resource": "student_score",
  "action": "view",
  "condition": {
    "program": { "not_in": ["gifted", "private"] }
  }
}
```

> Quyền này từ chối truy cập điểm số của học sinh thuộc chương trình đặc biệt.

---

### 🔄 Toán tử logic hỗ trợ (giả định)

| Toán tử      | Mô tả ví dụ                                                    |
| ------------ | -------------------------------------------------------------- |
| `eq`         | `\"campus\": \"HCM\"`                                          |
| `in`         | `\"subject_id\": [\"math\", \"phys\"]`                         |
| `not_in`     | `\"program\": { \"not_in\": [\"gifted\"] }`                    |
| `and` / `or` | Nested: `{ \"or\": [ {\"grade\": 9}, {\"campus\": \"HN\"} ] }` |
| `range`      | `{ \"score\": { \"gte\": 5, \"lte\": 8 } }`                    |

---

### 📘 Quy ước schema cho `condition`

* Mỗi permission `code` được ánh xạ tới một schema riêng biệt (do backend định nghĩa).
* API Gateway có một engine đánh giá `condition`, nhận:

  * `condition` từ DB
  * `context` từ request (VD: path params, query, JWT claim)
* Tên các key trong `condition` phải khớp với field trong `context`.

Ví dụ: Nếu request là `GET /students/{id}/score?term=HK1`, thì context có thể là:

```json
{
  "student_id": "stu-123",
  "term": "HK1",
  "user_id": "parent-456",
  "role": "parent"
}
```

---

> Để đơn giản hóa, các toán tử logic có thể được giới hạn hoặc hỗ trợ dần theo nhu cầu hệ thống.

---

## 6. Chiến lược cache RBAC & vô hiệu hóa

Để đảm bảo hiệu năng, tất cả permission của user được cache tại Redis dưới dạng:

```json
RBAC:{user_id} => {
  "role": "parent",
  "permissions": [
    {
      "code": "VIEW_SCORE_OWN_CHILD",
      "resource": "student_score",
      "action": "view",
      "condition": {
        "student_ids": ["stu-123"]
      }
    },
    {
      "code": "RECEIVE_NOTIFICATION",
      "resource": "notification",
      "action": "receive",
      "condition": null
    }
  ],
  "evaluated_at": "2025-05-10T13:00:00Z"
}
````
> ✅ API Gateway sẽ lấy danh sách này và evaluate từng condition tại thời điểm nhận request, sử dụng context như student_id, path, user_id, role, v.v.

---

### ⏱ TTL & cập nhật cache

* TTL mặc định: 5–15 phút
* API Gateway tự invalidate nếu:

  * Hết hạn TTL
  * Nhận sự kiện RBAC cập nhật

---

### 🔔 Cơ chế đồng bộ (propagation)

Khi có thay đổi liên quan đến:

* Role của user
* Permission gán cho role
* Trạng thái `is_active`

thì **User Service** phát sự kiện:

```json
{
  "type": "RBAC_UPDATED",
  "user_id": "u-456",
  "reason": "role_assigned"
}
```

Sự kiện này được publish qua Redis Pub/Sub hoặc message queue nội bộ.

---

### 📥 Gateway xử lý

* Subcribe vào `RBAC_UPDATED`
* Khi nhận sự kiện:

  * Xoá key `RBAC:{user_id}` khỏi Redis
  * Lần truy cập tiếp theo sẽ fetch lại từ User Service (lazy load)
  * Cache lại với TTL mới

---

### 🧯 Fallback khi cache miss

* Nếu cache miss hoặc lỗi Redis:

  * Gateway gọi API: `GET /users/{id}/permissions`
  * Áp dụng timeout & circuit breaker để tránh bottleneck
* Fallback đảm bảo system vẫn hoạt động chính xác → chỉ chậm hơn đôi chút

---

> Đây là mô hình hybrid: ưu tiên cache nhưng luôn có cơ chế fallback đảm bảo tính nhất quán.
  
  📎 Xem cấu trúc tổng thể và hạ tầng Redis/DB được sử dụng tại: 👉 [Deployment Overview Diagram – System Diagrams](./system-diagrams.md#9-deployment-overview-diagram--sơ-đồ-triển-khai-tổng-quan)

---

## 7. Hiệu năng & khả năng mở rộng

Cơ chế RBAC động tại API Gateway phải đảm bảo xử lý hàng nghìn request/giây mà không làm chậm hệ thống. Thiết kế hệ thống RBAC trong dx-vas đã được tối ưu theo các khía cạnh sau:

---

### ⚙️ Chiến lược đánh giá permission theo context

- Tất cả permissions được preload vào cache Redis, chỉ `condition` được evaluate tại runtime.
- Context được chuẩn hóa từ request (`student_id`, `class_id`, `role`, JWT claim...).
- Engine đánh giá `condition` được viết theo nguyên tắc:
  - Lightweight logic (tránh recursion, nested condition sâu)
  - Không gọi external API trong quá trình evaluate
  - Luôn fail-safe: nếu thiếu context → fail sớm

---

### 🚀 Tối ưu Redis Cache

- Redis key: `RBAC:{user_id}` → chứa danh sách permission object (code, resource, action, condition)
- TTL hợp lý: 5–15 phút, có thể cấu hình theo nhóm người dùng
- Dùng pipeline hoặc `MGET` nếu cần preload nhiều user
- Dữ liệu permission được serialize thành JSON đơn giản (không pickle)

---

### 📏 Benchmark hiệu năng (định hướng)

| Scenario                         | Trung bình (ms) | Notes |
|----------------------------------|------------------|-------|
| Evaluate 5 permissions (2 có condition) | 1–3 ms          | Trên Gateway |
| Redis cache hit                  | ~0.5 ms          | Dưới 1ms hầu hết các trường hợp |
| Redis cache miss (fallback DB)  | 20–40 ms         | Gọi `GET /users/{id}/permissions` |
| Evaluate fail (missing context) | < 1 ms           | Fail fast |

---

### 📈 Mở rộng theo quy mô

| Thành phần     | Mở rộng theo chiều ngang | Ghi chú |
|----------------|--------------------------|---------|
| API Gateway    | ✅                        | Stateless, autoscale Cloud Run |
| Redis          | ✅ (Redis Cluster)        | Shard theo `user_id` nếu cần |
| User Service   | ✅                        | Có thể scale theo traffic / RBAC API |
| Admin Webapp   | ✅                        | Tách frontend & API backend riêng |
| Pub/Sub        | ✅                        | RBAC_UPDATED event có thể scale broadcast |

> ⚠️ Backend không cần kiểm tra RBAC lại, giúp giảm tải đáng kể cho Redis và DB.

---

## 8. Bảo mật chuyên sâu cho RBAC

RBAC không chỉ là logic phân quyền mà còn là lớp kiểm soát truy cập quan trọng nhất trong hệ thống. Hệ thống dx-vas đã thiết kế nhiều lớp bảo vệ như sau:

---

### 🔐 Bảo vệ headers định danh

- Gateway sinh `X-User-ID`, `X-Role`, `X-Permissions`, `X-Auth-Method` sau khi xác thực
- Các header được:
  - Ký bằng `X-Signature` (HMAC hoặc JWT)
  - Hoặc chỉ chuyển trong môi trường nội bộ tin cậy (mTLS)
- Backend KHÔNG tự sinh hoặc chấp nhận header từ bên ngoài

---

### 🔒 Bảo vệ dữ liệu RBAC

- Bảng `permissions`, `roles`, `user_role`, `role_permission`:
  - Chỉ người có quyền `rbac:admin` mới chỉnh sửa
  - Toàn bộ thao tác được audit
- Không cho phép CRUD permission động qua frontend
  - Permission phải được migrate từ tệp YAML/tập tin cấu hình có kiểm soát
- Condition JSONB được validate schema kỹ trước khi ghi vào DB

---

### 🛡️ Phòng chống đặc quyền vượt mức (privilege escalation)

- Không cho phép người dùng tự gán role hoặc thay đổi role của người khác
- API cập nhật RBAC được bảo vệ bởi double-check: Gateway + Backend xác thực
- Một số quyền “nhạy cảm” được đánh dấu `protected: true` → cần phê duyệt trước khi gán

---

### 🔍 Giám sát hành vi bất thường

- Log:
  - `permission_evaluation_failed`
  - `unauthorized_access_attempt`
  - `rbac_modified_by` với trace ID
- Alert:
  - Người dùng bị từ chối liên tiếp 5+ lần trong 1 phút
  - RBAC change bất thường (ví dụ: 100 permission được gán trong 30s)

> ✅ Hệ thống RBAC được coi là tài sản bảo mật cấp cao, được giám sát ngang với Auth Service.

---

## 9. Giám sát & gỡ lỗi RBAC

RBAC là lớp kiểm soát truy cập quan trọng và cũng là điểm người dùng dễ gặp sự cố (“không thấy nút này”, “không truy cập được báo cáo nọ”). Việc giám sát, log và hỗ trợ debug RBAC là không thể thiếu.

---

### 🐛 Tình huống thường gặp

| Tình huống | Mô tả | Cách xử lý |
|-----------|-------|------------|
| Người dùng không thấy tính năng A | Có thể thiếu permission hoặc `condition` fail | Kiểm tra `X-Permissions` trong header |
| Người dùng truy cập bị lỗi 403 | Permission không hợp lệ hoặc JWT expired | Kiểm tra log RBAC evaluation |
| Người dùng có quá nhiều quyền | Sai role gán, thiếu condition | Audit lại `user_role`, `role_permission` |
| Quyền bị thu hồi nhưng vẫn truy cập được | Cache chưa invalidate | Xem thời điểm `RBAC_UPDATED` hoặc TTL |

---

### 🧪 Các log quan trọng cần bật

- `permission_evaluation_failed`: gồm `user_id`, `code`, `reason`, `context_missing`
- `permission_evaluation_passed`: tracking tần suất sử dụng quyền
- `rbac_cache_miss`: phát hiện vấn đề đồng bộ
- `rbac_modified_by`: audit trail chi tiết

> Tất cả log nên được gắn `Trace-ID` để truy ngược flow request.

---

### 📊 Metric theo dõi qua monitoring

- Tổng số `permission evaluations/s`
- Số lượng `evaluation failed` theo `code`
- Tỷ lệ `403` phân quyền / tổng `403`
- RBAC cache hit rate
- Sự kiện RBAC_UPDATED / phút

---

### 🔍 Quy trình debug điển hình

1. Bật dev mode trong Admin Webapp để in `X-Permissions`
2. Dùng Trace ID để tìm request trong log của Gateway
3. Xem log evaluate: lý do pass/fail
4. Dùng API Admin để xem `user -> role -> permission`
5. Kiểm tra `condition` và `context` cụ thể
6. Nếu cần, gọi API `GET /users/{id}/permissions?debug=true`

---

> ✅ Quan điểm vận hành: Phân quyền nên **debug được như bug logic**.

---

## 10. Best Practices trong quản trị role/permission

Một hệ thống RBAC mạnh nhưng cũng cần dễ quản trị và không trở thành “ma trận quyền lực”. Dưới đây là các thực hành được rút ra từ triển khai dx-vas.

---

### 🧭 Nguyên tắc tổ chức vai trò

- Mỗi `role` nên thể hiện 1 “chức năng thực tế” (giáo viên bộ môn, kế toán viên…)
- Tránh `role` quá tổng quát (admin, manager) nếu có quyền phức tạp
- Hạn chế gán nhiều `role` cho cùng 1 người – nếu cần thì phải **explicit**

---

### 🔐 Permission rõ ràng, có mã và mô tả

- Mỗi `permission` nên có:
  - `code`: duy nhất, ngắn gọn (VD: `EDIT_SCORE`, `VIEW_REPORT`)
  - `description`: mô tả nghiệp vụ
- Nên chuẩn hóa theo: `ACTION_RESOURCE_SCOPE` (VD: `VIEW_SCORE_OWN_CHILD`)
- Không dùng `permission` kiểu “wildcard” (`*:*`), trừ khi trong role `super_admin`

---

### 🧩 Điều kiện JSONB

- Ưu tiên preset (giao diện chọn lớp, chọn campus…) thay vì yêu cầu gõ tay
- Có schema riêng cho từng loại permission
- Nên review lại định kỳ các permission có condition phức tạp

---

### 📝 Audit & review định kỳ

- 3–6 tháng nên:
  - Review danh sách user có `admin` hoặc quyền `protected`
  - So sánh quyền thực tế với tài liệu nghiệp vụ
  - Xoá quyền không dùng (theo log `evaluation_passed` = 0 trong 90 ngày)

---

### 🔄 Quy trình cập nhật quyền

1. BTV nghiệp vụ gửi yêu cầu permission mới
2. Kỹ thuật đề xuất code, condition, scope
3. Review bảo mật nếu là quyền `write/delete` hoặc có `condition = null`
4. Nếu duyệt, cập nhật permission.yml + migrate DB
5. Gán cho role liên quan → propagate RBAC cache

---

> 🎯 Mục tiêu: quyền phải “đúng người – đúng nơi – đúng lúc”, và dễ hiểu với cả kỹ thuật lẫn nghiệp vụ.

---

## 11. Công cụ quản trị

* Admin Webapp sẽ có giao diện:

  * Tạo Role
  * Gán Permission
  * Gán Role cho User
  * Thiết lập Condition bằng UI (preset hoặc viết JSON trực tiếp)
  * Xem Audit Log theo user/time/resource

---

## 12. Tài liệu liên quan

* [ADR-006: Auth Strategy](../ADR/adr-006-auth-strategy.md)
* [ADR-007: RBAC Dynamic](../ADR/adr-007-rbac.md)
* [ADR-008: Audit Logging](../ADR/adr-008-audit-logging.md)
* [ADR-012: Response Structure](../ADR/adr-012-response-structure.md)

---

> “Phân quyền đúng – là chìa khóa mở ra đúng cánh cửa, đúng người, đúng thời điểm.”
