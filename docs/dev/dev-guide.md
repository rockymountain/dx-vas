# 📘 Dev Guide – dx-vas

Tài liệu này cung cấp hướng dẫn phát triển hệ thống dx-vas cho toàn bộ đội ngũ kỹ thuật. Bao gồm cách cài đặt môi trường, quy ước code, cấu trúc dịch vụ, quản lý RBAC, CI/CD, và các quy trình vận hành liên quan.

---

## Mục lục

1. [Giới thiệu tổng quan](#1-giới-thiệu-tổng-quan)
2. [Cài đặt môi trường phát triển](#2-cài-đặt-môi-trường-phát-triển)
3. [Cấu trúc thư mục & dịch vụ](#3-cấu-trúc-thư-mục--dịch-vụ)
4. [Quy ước viết mã & Coding Style](#4-quy-ước-viết-mã--coding-style)
5. [RBAC – Phân quyền động](#5-rbac--phân-quyền-động)
6. [Thiết kế API & OpenAPI](#6-thiết-kế-api--openapi)
7. [Gửi thông báo (Notification)](#7-gửi-thông-báo-notification)
8. [Quy trình test & CI/CD](#8-quy-trình-test--ci-cd)
9. [Migration & Quản lý cơ sở dữ liệu](#9-migration--quản-lý-cơ-sở-dữ-liệu)
10. [Quy trình pull request & review](#10-quy-trình-pull-request--review)
11. [Theo dõi & vận hành](#11-theo-dõi--vận-hành)
12. [Troubleshooting – Lỗi thường gặp](#12-troubleshooting--lỗi-thường-gặp)

---

## 1. Giới thiệu tổng quan

* Hệ thống dx-vas bao gồm các thành phần:

  * API Gateway (FastAPI)
  * Core Services: Auth Service, User Service, Notification Service
  * Business Adapters: CRM Adapter, SIS Adapter, LMS Adapter
  * Các thành phần hạ tầng: Redis, PostgreSQL, MySQL, Pub/Sub, GCS
* Kiến trúc microservices, tất cả triển khai trên Google Cloud Run
* Mọi truy cập đều thông qua API Gateway với kiểm soát RBAC động
* Sơ đồ hệ thống đầy đủ tại: [System Diagrams](../architecture/system-diagrams.md)

## 2. Cài đặt môi trường phát triển

Mục tiêu: giúp developer khởi tạo môi trường phát triển cho từng service trong hệ thống dx-vas, trong mô hình multi-repo (mỗi service là một Git repository riêng).

---

### 🧰 Yêu cầu hệ thống

- Python 3.11+ (cho các backend service)
- Node.js 18+ (cho frontend SPA/PWA)
- Docker + Docker Compose
- `direnv` (tuỳ chọn – tiện cho việc nạp biến môi trường)
- VS Code (khuyến nghị)

---

### 📁 Cấu trúc hệ thống (multi-repo)

Mỗi service là **một repo Git riêng biệt**:

```

📦 dx-api-gateway
📦 dx-auth-service
📦 dx-user-service
📦 dx-notification-service
📦 dx-crm-adapter
📦 dx-sis-adapter
📦 dx-lms-adapter
📦 dx-admin-webapp
📦 dx-customer-portal
📦 dx-vas

````

Tài liệu `dev-guide.md` này nằm trong repo `dx-vas`.

---

### 🔗 Cloning các repo liên quan

```bash
git clone git@github.com:vas/dx-api-gateway.git
git clone git@github.com:vas/dx-auth-service.git
git clone git@github.com:vas/dx-user-service.git
git clone git@github.com:vas/dx-notification-service.git
...
````

---

### 🔄 Cách chạy một service (VD: dx-auth-service)

```bash
cd dx-auth-service
cp .env.example .env

# Cài virtualenv và cài dependencies
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt

# Chạy service
uvicorn main:app --reload --port 8001
```

---

### 🔁 Cách chạy bằng Docker (tuỳ service)

```bash
cd dx-auth-service
docker build -t dx-auth .
docker run --env-file .env -p 8001:8000 dx-auth
```

---

### 🧪 Port mặc định từng service (dev)

| Service              | Port | Docs                                                     |
| -------------------- | ---- | -------------------------------------------------------- |
| Auth Service         | 8001 | [http://localhost:8001/docs](http://localhost:8001/docs) |
| User Service         | 8002 | [http://localhost:8002/docs](http://localhost:8002/docs) |
| Notification Service | 8003 | [http://localhost:8003/docs](http://localhost:8003/docs) |
| API Gateway          | 8080 | [http://localhost:8080/docs](http://localhost:8080/docs) |

---

📎 Tham khảo sơ đồ kết nối các service tại: [System Diagrams](../architecture/system-diagrams.md)

📎 Tham khảo cách thiết kế IC cho từng service tại: [Interface Contracts](../interfaces/adr-index.md)

## 3. Cấu trúc thư mục & dịch vụ

Hệ thống dx-vas sử dụng kiến trúc microservices theo mô hình **multi-repo**, trong đó mỗi service là một repository độc lập, triển khai riêng biệt.

---

### 🧱 Phân loại các repo

| Nhóm | Repo | Mô tả |
|------|------|------|
| Gateway | `dx-api-gateway` | Cổng vào duy nhất, kiểm tra phân quyền (RBAC) |
| Core Services | `dx-auth-service`, `dx-user-service`, `dx-notification-service` | Các service lõi chạy độc lập |
| Business Adapters | `dx-crm-adapter`, `dx-sis-adapter`, `dx-lms-adapter` | Adapter kết nối đến hệ thống CRM/SIS/LMS bên ngoài |
| Frontend | `dx-admin-webapp`, `dx-customer-portal` | Giao diện cho admin (SPA) và phụ huynh/học sinh (PWA) |
| Tài liệu | `dx-docs` | Chứa toàn bộ tài liệu kỹ thuật, sơ đồ, hướng dẫn |

---

### 📦 Gợi ý cấu trúc thư mục nội bộ (mỗi service backend)

Áp dụng cho tất cả backend service (Python + FastAPI):

```

dx-<service>/
├── app/                    # Source code chính
│   ├── main.py             # Entry point FastAPI
│   ├── api/                # Routers
│   ├── models/             # SQLAlchemy models
│   ├── schemas/            # Pydantic schemas
│   ├── services/           # Business logic
│   └── core/               # Cấu hình, security, middleware
├── tests/                  # Unit & integration tests
├── alembic/                # Migration (nếu dùng PostgreSQL)
├── .env.example            # Mẫu biến môi trường
├── requirements.txt        # Dependencies
└── README.md

```

---

### 🧩 Giao tiếp giữa các service

- Mọi request từ frontend hoặc service khác đều đi qua **API Gateway**
- RBAC được kiểm tra tại Gateway trước khi forward request
- Redis được dùng để cache RBAC và session token
- Giao tiếp bất đồng bộ (event) sử dụng Pub/Sub

---

### 📄 Gợi ý chuẩn hóa template

- Các service nên khởi tạo từ cùng một **service template chuẩn** để đảm bảo:
  - Cấu trúc đồng nhất
  - Hệ thống logging/tracing thống nhất
  - Cùng hệ thống health check, swagger, RBAC header, metrics

> 📎 Template tham khảo: [`dx-service-template`](https://github.com/vas/dx-service-template) *(giả định)*

---

📎 Sơ đồ luồng giao tiếp các service có thể xem tại: [System Diagrams](../architecture/system-diagrams.md)

## 4. Quy ước viết mã & Coding Style

Quy ước này áp dụng cho toàn bộ backend services và frontend apps trong hệ thống dx-vas nhằm đảm bảo:

- Đồng nhất về style
- Dễ đọc, dễ review
- Dễ bảo trì và mở rộng

---

### 🐍 Python – Backend Services

Áp dụng cho các repo: `dx-auth-service`, `dx-user-service`, `dx-notification-service`, và các adapter.

#### ✅ Coding style

- Tuân theo chuẩn PEP8
- Sử dụng `black` để auto-format
- Import theo nhóm (stdlib, third-party, local)

#### 📦 Tool

```bash
pip install black isort flake8
````

#### 🔁 Format code

```bash
black app/
isort app/
flake8 app/
```

#### 📘 Gợi ý cấu trúc project

```
app/
├── main.py            # Entry point FastAPI
├── api/               # Routers (tổ chức theo domain)
├── models/            # SQLAlchemy models
├── schemas/           # Pydantic DTOs
├── services/          # Business logic
├── core/              # Cấu hình, bảo mật, middleware
└── utils/             # Hàm dùng chung
```

---

### 🌐 JavaScript/TypeScript – Frontend (SPA & PWA)

Áp dụng cho các repo: `dx-admin-webapp`, `dx-customer-portal`

#### ✅ Coding style

* Sử dụng TypeScript toàn bộ
* Tuân theo Airbnb style guide (hoặc ESLint chuẩn Vite/Next)
* Format bằng Prettier

#### 📦 Tool

```bash
npm install --save-dev eslint prettier
```

#### 🔁 Format code

```bash
npx prettier --write .
npx eslint . --fix
```

---

### 🔍 Tên biến, tên hàm, tên file

| Loại         | Format      | Ví dụ                                |
| ------------ | ----------- | ------------------------------------ |
| Biến Python  | snake\_case | `access_token`, `is_active`          |
| Class Python | PascalCase  | `UserService`, `PermissionCondition` |
| Route        | kebab-case  | `/reset-password`                    |
| File TS      | kebab-case  | `login-form.tsx`                     |

---

### 🧩 RBAC & Permission Naming

Tuân theo cấu trúc: `RESOURCE_ACTION[_SCOPE]`
Ví dụ:

* `VIEW_SCORE`
* `EDIT_SCORE_OWN_CLASS`
* `RECEIVE_NOTIFICATION_PARENT`

> 📎 Chi tiết tại: [RBAC Deep Dive](../architecture/rbac-deep-dive.md#5-permission-có-điều-kiện-condition-jsonb)

---

### ✅ Commit Message Convention

Áp dụng format `type(scope): message`

Ví dụ:

```
feat(auth): add OTP fallback logic
fix(user): wrong default role assignment
refactor: move shared utils to core/
```

Các loại type phổ biến:

* `feat`, `fix`, `refactor`, `docs`, `test`, `chore`, `perf`

---

📎 Xem ví dụ cấu trúc chuẩn tại: [`dx-service-template`](https://github.com/vas/dx-service-template) *(giả định)*

## 5. RBAC – Phân quyền động

Hệ thống dx-vas sử dụng mô hình **RBAC động (Contextual Role-Based Access Control)**, trong đó quyền được đánh giá theo vai trò, hành động và điều kiện `condition` tùy ngữ cảnh.

---

### ⚙️ Cấu trúc RBAC

| Thành phần | Mô tả |
|------------|-------|
| `Role` | Vai trò người dùng (admin, teacher, parent...) |
| `Permission` | Một hành động cụ thể lên một tài nguyên |
| `Condition (JSONB)` | Điều kiện kiểm soát ngữ cảnh – áp dụng theo user, resource, thời điểm |
| `User-Role Mapping` | Một user có thể gán nhiều vai trò |
| `Role-Permission Mapping` | Mỗi vai trò gán nhiều permission |

📘 Các bảng được mô tả trong: [RBAC Deep Dive](../architecture/rbac-deep-dive.md#4-cấu-trúc-dữ-liệu-rbac)

---

### 🔐 Quy tắc đánh giá phân quyền

1. API Gateway nhận request → kiểm tra token (JWT)
2. Trích `user_id`, `role`, và context (VD: `student_id`, `class_id`, ...)
3. Tìm danh sách permission của user từ Redis cache (hoặc gọi `User Service`)
4. Với mỗi permission có `condition`, evaluate theo `context`
5. Nếu có ít nhất 1 permission thỏa điều kiện → allow request

---

### 📋 Đặt tên permission

Tuân theo format:

```

<RESOURCE>*<ACTION>\[*<SCOPE>]

````

Ví dụ:

- `VIEW_SCORE_OWN_CHILD`
- `EDIT_SCORE_CLASS_OWNER`
- `RECEIVE_NOTIFICATION_PARENT`

> 📎 Chi tiết tại: [RBAC Deep Dive](../architecture/rbac-deep-dive.md#5-permission-có-điều-kiện-condition-jsonb)

---

### 📦 Cách định nghĩa permission mới

1. Thêm dòng mới vào file YAML `permissions.yaml` trong repo `dx-user-service`
2. Chạy lệnh migrate RBAC:

```bash
make rbac-migrate
````

3. Gán permission này vào role trong bảng `role_permission`
4. Gán role cho user → tự động invalidate cache qua sự kiện `rbac_updated`

---

### 🔍 Gợi ý khi thiết kế condition

* Dùng khóa rõ ràng: `class_ids`, `student_ids`, `term`, ...
* Tránh condition quá phức tạp → phân tách thành nhiều permission đơn giản
* Test với context thực tế qua API Gateway

---

### 🧪 Test RBAC

* Mock user và role
* Gọi API thật qua Gateway với JWT hợp lệ
* Kiểm tra log: permission granted / denied
* Kiểm tra hành vi với user bị `is_active = false`

---

📎 Xem sơ đồ RBAC chi tiết tại: [System Diagrams](../architecture/system-diagrams.md#4-rbac-evaluation-flow--luồng-đánh-giá-phân-quyền-động)

## 6. Thiết kế API & OpenAPI

Toàn bộ hệ thống dx-vas tuân theo nguyên tắc API chuẩn RESTful, được định nghĩa bằng **OpenAPI 3.0.3** (sử dụng FastAPI).

---

### 📐 Nguyên tắc thiết kế API

| Tiêu chí | Quy định |
|----------|----------|
| URL | lowercase, dùng kebab-case |
| Method | Tuân chuẩn REST: GET/POST/PATCH/DELETE |
| Versioning | `v1` gắn vào prefix: `/v1/users/...` |
| Response | Gói trong `data`, `meta`, `error` |
| Error | Dùng mã lỗi chuẩn, message rõ ràng |
| RBAC | Header `X-Permissions`, context dynamic |

---

### 📦 Cấu trúc response chuẩn

```json
{
  "data": {...},
  "meta": {
    "request_id": "uuid",
    "timestamp": "2024-01-01T12:00:00Z"
  },
  "error": null
}
````

Khi lỗi:

```json
{
  "data": null,
  "meta": {...},
  "error": {
    "code": "USER_NOT_FOUND",
    "message": "User does not exist"
  }
}
```

---

### 📘 Định nghĩa API bằng OpenAPI

* Mỗi service có một file `openapi.yaml` định nghĩa OpenAPI 3.0.3
* Có thể generate tự động từ FastAPI hoặc viết tay với VS Code plugin
* Dùng Redoc, Swagger UI hoặc Stoplight Studio để xem

---

### 📁 Ví dụ repo `dx-user-service`

```
dx-user-service/
├── app/
│   ├── api/
│   │   └── users.py         # Define @router.get("/users")
├── docs/
│   └── openapi.yaml         # OpenAPI 3.0.3 schema
```

---

### ✅ API Gateway routing

* API Gateway định tuyến đến các service theo rule
* Mỗi route có thể được gắn với:

  * `target_service`
  * `required_permissions`
  * `path_rewrite` (nếu cần)

---

### 🧪 Kiểm thử API

* Gọi trực tiếp local (FastAPI): [http://localhost:8001/docs](http://localhost:8001/docs)
* Gọi qua API Gateway (RBAC enforced): [http://localhost:8080/v1/](http://localhost:8080/v1/)...
* Dùng Postman, HTTPie, Swagger UI, hoặc test script tự động

---

📎 Xem Interface Contracts (ICs) tại: [`docs/interfaces/`](../interfaces/)

📎 Định dạng chuẩn response/error được mô tả tại: [`adr-001-fastapi`](../ADR/adr-001-fastapi.md) *(giả định)*

## 7. Gửi thông báo (Notification)

Notification Service là một core service trong dx-vas, chịu trách nhiệm gửi thông báo đến người dùng qua nhiều kênh khác nhau: Web, Email, Zalo OA, Google Chat.

---

### 📬 Các kênh hỗ trợ

| Kênh | Đối tượng chính | Ghi chú |
|------|-----------------|---------|
| WebPush | Học sinh, giáo viên | Nhận trong Admin Webapp hoặc PWA |
| Gmail API | Phụ huynh, giáo viên | Gửi email qua Google Workspace |
| Zalo ZNS | Phụ huynh | Yêu cầu đăng ký OA + duyệt template |
| Google Chat | Nhân viên nội bộ | Dành cho thông báo hệ thống, vận hành |

---

### 📦 Các API chính

> Tất cả API đều gọi qua API Gateway  
> Gateway kiểm tra quyền `RECEIVE_NOTIFICATION_*`

#### `POST /notifications/send`

Gửi thông báo theo template:

```json
{
  "channel": "zalo",
  "user_id": "u123",
  "template_id": "ZALO_FEE_REMINDER",
  "payload": {
    "amount": "2.000.000",
    "due_date": "2024-01-10"
  }
}
````

#### `POST /notifications/register-channel`

Gán user với thông tin thiết bị (device token, Zalo ID, email...)

---

### 📡 Gửi thông báo qua sự kiện (event-driven)

* Các service như CRM/SIS/LMS có thể phát sự kiện `NOTIFICATION_REQUESTED`
* Notification Service subscribe từ Pub/Sub
* Ví dụ event:

```json
{
  "event": "NOTIFICATION_REQUESTED",
  "user_id": "u123",
  "channel": "email",
  "template_id": "WELCOME",
  "payload": {
    "name": "Nguyễn Văn A"
  }
}
```

---

### 🔄 Ưu tiên & fallback kênh

* Mỗi user có thể cấu hình kênh ưa thích (stored in User Service)
* Nếu gửi thất bại (timeout/quota), có thể fallback sang email

---

### 🧪 Test Notification Service

* Gọi thử API `/send` với JWT hợp lệ
* Kiểm tra log gửi thành công/thất bại
* Giả lập sự kiện trên Pub/Sub
* Kiểm tra trạng thái từng message qua Admin Webapp

---

📎 Xem sơ đồ Notification Flow tại: [System Diagrams](../architecture/system-diagrams.md#3-notification-flow--luồng-gửi-thông-báo)

📎 Định nghĩa chi tiết tại IC: [`ic-04-notification.md`](../interfaces/ic-04-notification.md)

## 8. Quy trình test & CI/CD

Hệ thống dx-vas áp dụng quy trình kiểm thử và triển khai liên tục cho từng service độc lập. Mỗi service có workflow CI/CD riêng, chạy tự động trên GitHub Actions hoặc Google Cloud Build.

---

### ✅ Các tầng kiểm thử

| Loại test | Mục tiêu | Ví dụ |
|-----------|----------|-------|
| Unit test | Kiểm tra logic đơn vị | Hàm xử lý OTP, parse template |
| Integration test | Kiểm tra nhiều module kết hợp | Gọi API `/users` → trả response đúng |
| E2E test | Kiểm tra end-to-end toàn hệ thống (tùy chọn) | Đăng ký học sinh từ CRM → đồng bộ qua SIS, LMS |

---

### 📦 Công cụ

- `pytest`, `pytest-cov`: cho Python services
- `requests`, `httpx`: test API
- `docker-compose.test.yml`: khởi chạy môi trường test có DB/Redis
- `testcontainers`: nếu muốn test với Redis/PostgreSQL thật

---

### 🧪 Chạy test local

```bash
# Trong từng repo service
pytest tests/
pytest --cov=app tests/
````

---

### 📊 Yêu cầu coverage

* Core logic (RBAC, Auth, Notification): ≥ 85%
* Adapter/API layer: ≥ 70%

> CI sẽ fail nếu coverage thấp hơn ngưỡng quy định

---

### 🧱 CI Workflow (GitHub Actions)

```yaml
name: CI

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      - run: pip install -r requirements.txt
      - run: pytest --cov=app tests/
```

---

### 🚀 Triển khai (CD)

* Dùng Google Cloud Build / GitHub Actions
* Triển khai Cloud Run theo branch/tag:

| Branch | Môi trường | Ghi chú          |
| ------ | ---------- | ---------------- |
| `dev`  | Staging    | Auto-deploy      |
| `main` | Production | Require approval |

---

### 🛠 Makefile chuẩn

Mỗi repo nên có:

```make
test:
    pytest --cov=app tests/

format:
    black app/ && isort app/

run:
    uvicorn app.main:app --reload --port=8001
```

---

📎 Xem ADR CI/CD tại: [`adr-001-ci-cd.md`](../ADR/adr-001-ci-cd.md)

## 9. Migration & Quản lý cơ sở dữ liệu

Mỗi service trong dx-vas quản lý database riêng biệt, không chia sẻ DB trực tiếp. Core Services dùng PostgreSQL, Adapters dùng MySQL. Mọi thay đổi dữ liệu đều được quản lý qua migration có version.

---

### 🗄️ Công cụ

| Loại DB | Công cụ | Ghi chú |
|---------|---------|--------|
| PostgreSQL | Alembic | Dùng chung với SQLAlchemy |
| MySQL | Alembic hoặc raw SQL | Tùy dịch vụ |

---

### 📦 Cấu trúc thư mục

```

dx-user-service/
├── alembic/
│   ├── versions/
│   └── env.py
├── alembic.ini

````

---

### 🧪 Tạo migration mới

```bash
alembic revision -m "add table permission"
````

Sau đó sửa file trong `versions/` để định nghĩa lệnh SQL.

---

### 🚀 Chạy migration

```bash
alembic upgrade head
```

> ⚠️ Cần chắc chắn kết nối `DATABASE_URL` đã đúng (env/dev.env)

---

### 🔁 Rollback migration

```bash
alembic downgrade -1
```

---

### ✅ Đặt tên migration chuẩn

Format: `yymmdd_hhmm_<slug>.py`

Ví dụ: `240112_1030_add_role_table.py`

---

### 🧩 Quản lý schema chung

* Core schema như `Permission`, `Role`, `User` được chuẩn hóa và migrate từ `dx-user-service`
* Không sửa trực tiếp dữ liệu RBAC trong DB → dùng migration hoặc API quản trị
* Dữ liệu ban đầu (seed) được load qua script Python hoặc fixture `.sql`

---

### 🧪 Test DB local

```bash
docker run --rm -e POSTGRES_PASSWORD=dx -p 5432:5432 postgres:15
```

Hoặc dùng Docker Compose trong `dev` môi trường

---

📎 Xem định nghĩa schema chuẩn tại: [RBAC Deep Dive](../architecture/rbac-deep-dive.md#4-cấu-trúc-dữ-liệu-rbac)

## 10. Quy trình pull request & review

Mỗi thay đổi trong hệ thống dx-vas phải được thực hiện thông qua Pull Request (PR) theo quy trình rõ ràng và có kiểm duyệt. Quy trình này đảm bảo:

- Code rõ ràng, đúng chuẩn style
- Logic được test và review
- Không ảnh hưởng đến hệ thống vận hành

---

### 🚀 Quy trình phát triển cơ bản

1. **Tạo nhánh mới từ `dev`**

```bash
git checkout dev
git pull
git checkout -b feat/<service>-<short-desc>
````

2. **Viết code & test**
3. **Chạy test + format code**
4. **Push lên GitHub**
5. **Tạo PR về `dev`**

---

### ✅ Quy định đặt tên nhánh

```
feat/<service>-ten-tinh-nang
fix/<service>-ten-loi
refactor/<service>-ten-module
```

Ví dụ:

* `feat/auth-otp-login`
* `fix/user-role-sync-error`

---

### 🧩 Checklist khi mở PR

* [ ] Đã viết unit test và integration test (nếu cần)
* [ ] Đã chạy `black`, `isort`, `flake8` (Python)
* [ ] Đã chạy `prettier`, `eslint` (Frontend)
* [ ] Không push `.env`, credential, dữ liệu cá nhân
* [ ] Không sửa file cấu hình deployment production trừ khi được duyệt

---

### 👀 Reviewer cần kiểm tra

* Logic có hợp lý không?
* Tên biến, tên hàm có rõ ràng?
* Có gây side effect không mong muốn?
* Đã có test & chạy pass chưa?
* Có ảnh hưởng đến schema / RBAC không? (→ cần update migration hoặc IC?)

---

### 🛡️ Luồng PR điển hình

```
Dev → Push lên nhánh feat/... → Mở PR → Review + Comment → Sửa đổi → Merge
```

---

### 🔒 Luật merge

| Môi trường | Nhánh  | Merge vào | Ghi chú                  |
| ---------- | ------ | --------- | ------------------------ |
| Staging    | `dev`  | `dev`     | Merge sau khi pass CI    |
| Production | `main` | `main`    | Require approval 2 người |

---

📎 Xem CI/CD detail: [`adr-001-ci-cd.md`](../ADR/adr-001-ci-cd.md)

📎 Xem Coding Style tại: [Mục 4 – Quy ước viết mã](#4-quy-ước-viết-mã--coding-style)

## 11. Theo dõi & vận hành

Hệ thống dx-vas được triển khai theo kiến trúc phân tán, yêu cầu giám sát chủ động và khả năng truy vết lỗi hiệu quả. Tất cả service đều chạy trên Google Cloud Run và tích hợp logging/tracing tập trung.

---

### 📦 Thành phần quan sát chính

| Thành phần | Mục tiêu |
|------------|----------|
| Cloud Logging | Tập trung log từ tất cả container |
| Cloud Monitoring | Giám sát CPU, latency, error rate |
| Cloud Trace | Theo dõi trace request phân tán |
| Cloud Alerting | Thiết lập cảnh báo theo rule |
| Pub/Sub Dead Letter | Ghi nhận sự kiện xử lý lỗi / timeout |

---

### 🛠 Mỗi service cần implement

- Logging chuẩn hóa theo JSON:
  ```json
  {
    "level": "INFO",
    "service": "user-service",
    "request_id": "abc-123",
    "message": "User updated",
    "user_id": "u789"
  }
````

* Trace header (`X-Request-ID`) cần được truyền xuyên suốt mọi service
* Bắt timeout, quota error từ external API (Zalo, Gmail...) → log cảnh báo

---

### 🔐 Log bảo mật

* Log RBAC: `permission_denied`, `rbac_modified_by`, `invalid_token`
* Log auth: `login_failed`, `otp_failed`, `user_locked`
* Gửi alert khi:

  * Đăng nhập sai liên tiếp > 5 lần
  * Thay đổi quyền role đặc biệt
  * Truy cập endpoint nhạy cảm thất bại

---

### ⚠️ Alerting gợi ý

| Rule                | Điều kiện                             | Mức         |
| ------------------- | ------------------------------------- | ----------- |
| Gateway error spike | `5xx` tăng > 20% trong 5 phút         | ⚠️ Warning  |
| Pub/Sub failed ack  | `notification_event` timeout > 10 lần | 🚨 Critical |
| Redis miss rate cao | Miss rate > 40% trong 10 phút         | ⚠️ Warning  |
| Login fail          | 10+ lần từ 1 IP trong 1 phút          | 🚨 Security |

---

### 🧪 Health check & trace

* Tất cả service expose `/healthz` trả `200 OK`
* Gateway trace từng request → gắn `request_id`
* Pub/Sub và Redis có dashboard trạng thái riêng

---

📎 Xem thêm: [`adr-008-audit-logging.md`](../ADR/adr-008-audit-logging.md)

📎 Sơ đồ triển khai observability: [System Diagrams](../architecture/system-diagrams.md#9-deployment-overview-diagram--sơ-đồ-triển-khai-tổng-quan)

## 12. Troubleshooting – Lỗi thường gặp

Dưới đây là danh sách một số lỗi thường gặp trong quá trình phát triển, test hoặc vận hành hệ thống dx-vas – cùng với cách xử lý đề xuất.

---

### 🔑 Authentication & Token

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `401 Unauthorized` | Thiếu hoặc sai JWT | Kiểm tra `Authorization: Bearer <token>` |
| `Signature has expired` | Token hết hạn | Gọi lại `/refresh-token` |
| `403 Forbidden` | Không có quyền | Kiểm tra permission hoặc header `X-Permissions` |

---

### 🔐 RBAC & Redis

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `permission denied` | Không match `condition` | Debug bằng context gửi qua request |
| Cache không cập nhật | RBAC chưa được invalidate | Gọi lại API hoặc đợi TTL Redis (~5 phút) |
| `KeyError` RBAC | Sai format permission | Kiểm tra file `permissions.yaml` và migration |

---

### 🛠 Database & Migration

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `alembic error` | Thiếu `env.py` hoặc config | Kiểm tra `alembic.ini` & env var `DATABASE_URL` |
| `table does not exist` | Quên migrate | `alembic upgrade head` |
| `duplicate key` | Trùng dữ liệu seed | Kiểm tra file insert hoặc set `on_conflict` |

---

### 📦 Docker & Service Dev

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `address already in use` | Port bị chiếm | Đổi port trong `.env` hoặc docker |
| `ModuleNotFoundError` | Thiếu dependency | `pip install -r requirements.txt` hoặc rebuild container |
| Service không start | Lỗi env hoặc lỗi DB | Kiểm tra `.env`, log DB, Redis, Pub/Sub |

---

### 🧪 CI/CD & Test

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| `pytest ImportError` | Chạy ngoài virtualenv | `source .venv/bin/activate` |
| `coverage < 70%` | Thiếu test cho logic chính | Thêm unit test cho service, router, schema |
| CI fail ở `black` | Không format code | `black app/` rồi commit lại |

---

### 📡 Notification

| Lỗi | Nguyên nhân | Giải pháp |
|-----|-------------|-----------|
| Không gửi Zalo | Sai template hoặc quota | Kiểm tra token OA + cấu hình template ID |
| Gửi email thất bại | Lỗi Gmail API quota | Chờ retry hoặc contact admin |
| Push không nhận | User chưa đăng ký device token | Gọi `/notifications/register-channel` trước |

---

### 🧩 General Tips

- Sử dụng header `X-Request-ID` → dễ truy vết trong log
- Dùng `curl` hoặc `httpie` để test đơn giản
- Kiểm tra log Cloud Run hoặc container local nếu service crash
- Dùng `direnv`, `.env.example`, `.envrc` để tránh quên biến môi trường

---

📎 Xem log và trace tại: [System Diagrams](../architecture/system-diagrams.md#9-deployment-overview-diagram--sơ-đồ-triển-khai-tổng-quan)
  
📎 Vấn đề liên quan đến RBAC: [RBAC Deep Dive](../architecture/rbac-deep-dive.md)
