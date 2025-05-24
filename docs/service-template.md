# 📦 dx_vas – Service Template Structure

> Chuẩn hóa cấu trúc thư mục và tệp tin cho các service (Python FastAPI hoặc NodeJS) trong hệ thống dx_vas.

---

## 🧱 Cấu trúc thư mục đề xuất

```bash
my_service/
├── app/
│   ├── __init__.py
│   ├── main.py                  # entrypoint chính
│   ├── config/                  # mã nguồn xử lý cấu hình runtime (load từ biến môi trường)
│   │   ├── base.py
│   │   ├── dev.py
│   │   └── prod.py
│   ├── routers/                 # định nghĩa các route theo module
│   │   ├── __init__.py
│   │   ├── users.py
│   │   └── auth.py
│   ├── schemas/                 # Pydantic models
│   │   ├── __init__.py
│   │   ├── user.py
│   │   └── token.py
│   ├── services/                # business logic / adapter
│   │   ├── __init__.py
│   │   └── user_service.py
│   ├── repositories/           # logic truy cập dữ liệu (ORM, query...)
│   │   ├── __init__.py
│   │   └── user_repo.py
│   ├── utils/                   # các hàm tiện ích chung
│   │   ├── __init__.py
│   │   └── crypto.py
│   └── dependencies.py          # global dependency injection
├── tests/
│   ├── __init__.py
│   ├── conftest.py
│   ├── test_users.py
│   └── test_auth.py
├── config/
│   ├── base.env                 # Biến chung không nhạy cảm
│   ├── dev.env                  # Biến dành cho môi trường dev (không commit)
│   ├── staging.env              # Biến staging (không commit)
│   └── production.env           # Biến production (không commit)
├── .env                         # symbolic link hoặc biến chạy local
├── .env.example                 # template biến môi trường
├── Dockerfile                   # build container
├── Makefile                     # câu lệnh tiện ích: run, test, lint...
├── README.md                    # mô tả service
├── requirements.txt             # thư viện phụ thuộc
└── pyproject.toml               # cấu hình black, isort, v.v (tuỳ chọn)
```

> 🔗 Tham chiếu chuẩn từ [ADR-005](./ADR/adr-005-env-config.md):
> - `config/` ở gốc chứa các file `.env` cho từng môi trường.
> - `app/config/` chứa mã nguồn Python/NodeJS để load, parse, và ánh xạ các biến môi trường thành đối tượng settings.

> Với NodeJS, thay đổi `schemas/` → `types/` và dùng `tsconfig.json`, `package.json`, `src/`, `test/`, `dotenv`, `Jest`, `ESLint`, `Prettier`.

---

## ⚙️ Quy ước kỹ thuật

| Thành phần       | FastAPI                             | NodeJS                             |
|------------------|--------------------------------------|------------------------------------|
| HTTP Server      | `uvicorn app.main:app`              | `express`, `fastify`, `nestjs`     |
| Env config       | `pydantic.BaseSettings`             | `dotenv`, `envsafe`                |
| DI / Router      | FastAPI `Depends`, `APIRouter`      | Module system hoặc DI của NestJS  |
| Schema Validation| `Pydantic` models                   | `zod`, `Joi`, `class-validator`    |
| Testing          | `pytest` + `httpx` hoặc `requests`  | `jest`, `supertest`                |
| Lint + Format    | `black`, `flake8`, `isort`          | `eslint`, `prettier`               |

---

## 📄 README.md – Mẫu nội dung

```md
# 📘 My Service – dx_vas

## 🧩 Mục tiêu
Mô tả chức năng, phạm vi và nhóm phụ trách.

## 🚀 Khởi chạy local
```bash
make run
```

## 🧪 Test
```bash
make test
```

## 🛠 Cấu trúc thư mục
- `app/config/`: cấu hình runtime (load từ biến môi trường)
- `app/routers/`: route theo module
- `app/schemas/`: model dùng chung
- `app/repositories/`: thao tác dữ liệu (SQLAlchemy/ORM/...)

## 🔐 Secrets & Env
- `.env.example` chứa các biến môi trường mẫu
- Biến thực tế được chọn từ `config/{ENV}.env` (xem [ADR-005](./ADR/adr-005-env-config.md))

## 📎 Tài liệu liên quan
- [Interface Contract](./interfaces/my_service.md)
- [ADR liên quan](./ADR/README.md)
```

---

## 🧰 Makefile – Mẫu

```Makefile
run:
	uvicorn app.main:app --reload

test:
	pytest tests/ -v

lint:
	flake8 app/ tests/
	black app/ tests/ --check

format:
	black app/ tests/
	isort app/ tests/
```

---

## ✅ Checklist CI/CD

- [ ] Có `.env.example`, không commit `.env`
- [ ] Có test và linter trong CI (GitHub Actions hoặc GitLab CI)
- [ ] Có endpoint `/healthz` cho health check
- [ ] Có endpoint `/readyz` cho readiness probe (được dùng bởi Gateway hoặc load balancer)
- [ ] Có OpenAPI (`GET /openapi.json`) nếu là API service

---

> “Một service tốt bắt đầu từ cấu trúc rõ ràng và nhất quán.”
