# 🧭 ONBOARDING Checklist – API Gateway (DX VAS)

Chào mừng bạn đến với đội ngũ phát triển API Gateway của dự án **Chuyển đổi số VAS**! 🎉

Tài liệu này sẽ giúp bạn khởi động nhanh chóng, nắm bắt quy trình làm việc và thiết lập đầy đủ các công cụ, quyền truy cập để bắt đầu đóng góp hiệu quả.

---

## ✅ Checklist tuần đầu tiên

> **Gợi ý thời gian:**
>
> * **Ngày 1–2:** Truy cập hệ thống & Thiết lập môi trường local
> * **Ngày 3–4:** Làm quen codebase & công cụ CI/CD
> * **Ngày 5:** Giao tiếp, góp PR đầu tiên, chuẩn bị review

### 1. Truy cập & công cụ

* [ ] Được cấp quyền truy cập vào:

  * [ ] GitHub repo: `vas-org/api-gateway`
  * [ ] Google Cloud Console (staging project)
  * [ ] Slack nội bộ / nhóm DX VAS
* [ ] Cài đặt công cụ:

  * [ ] [Python >= 3.10](https://www.python.org/downloads/)
  * [ ] [Docker & Docker Compose](https://www.docker.com/products/docker-desktop/)
  * [ ] [Git](https://git-scm.com/)
  * [ ] [VSCode](https://code.visualstudio.com/) hoặc [PyCharm](https://www.jetbrains.com/pycharm/)
  * [ ] Redis client (TablePlus hoặc `redis-cli`)
  * [ ] PostgreSQL client ([DBeaver](https://dbeaver.io/), `psql`)

### 2. Thiết lập môi trường local

* [ ] Clone dự án và tạo file `.env`
* [ ] Cài đặt `pre-commit` và chạy `pre-commit install`
* [ ] `docker-compose up --build` và truy cập `http://localhost:8000/docs`
* [ ] Đảm bảo FastAPI hoạt động local không lỗi

### 3. Làm quen với mã nguồn và quy trình

* [ ] Đọc tài liệu [`DEV_GUIDE.md`](./DEV_GUIDE.md)
* [ ] Đọc quy ước [`CONTRIBUTING.md`](./CONTRIBUTING.md)
* [ ] Xem kiến trúc thư mục và module chính: `auth`, `rbac`, `notify`
* [ ] Chạy thử test: `pytest`
* [ ] Đọc CI workflow tại `.github/workflows/ci.yml`
* [ ] Làm quen với `prestart.sh` và Dockerfile

### 4. Giao tiếp & văn hóa nội bộ

* [ ] Tham gia nhóm Slack `#dx-vas-dev`
* [ ] Giới thiệu bản thân trong nhóm (đơn giản + role)
* [ ] Biết ai là mentor kỹ thuật (thường là người assign task đầu tiên)

---

## 🚧 Nhiệm vụ tuần đầu tiên (đề xuất)

| Mục tiêu          | Task mẫu                                | Kỳ vọng                              |
| ----------------- | --------------------------------------- | ------------------------------------ |
| Làm quen codebase | Fix lỗi nhỏ hoặc thêm log/debug         | 1 commit, push lên branch feature/\* |
| CI/CD             | Tạo PR nhỏ cập nhật README hoặc badge   | Tạo 1 PR & được merge vào `dev`      |
| Test              | Viết thêm 1 test cho route `auth/login` | Có thể chạy pass trong `pytest`      |

---

## 🧑‍🏫 Người hỗ trợ

| Hạng mục        | Người phụ trách                  |
| --------------- | -------------------------------- |
| Mentor kỹ thuật | Nguyễn Văn A (Tech Lead)         |
| Code review     | Bất kỳ dev cấp senior trong team |
| CI/CD & Cloud   | Trần Thị B (DevOps)              |

---

## 📎 Tài liệu quan trọng

* Dev Guide: [`docs/DEV_GUIDE.md`](./DEV_GUIDE.md)
* Coding Convention + PR flow: [`docs/CONTRIBUTING.md`](./CONTRIBUTING.md)
* Kiến trúc: `docs/ADR/adr-001-fastapi.md`, `adr-002-rbac-design.md`
* API Docs (local): [`http://localhost:8000/docs`](http://localhost:8000/docs)

---

## 💬 Giao tiếp và văn hóa phản hồi

* Bạn **không cần biết mọi thứ từ đầu** – hãy **hỏi khi chưa rõ**! Slack luôn mở.
* Feedback nên tập trung vào hành vi/code thay vì cá nhân.
* Đưa feedback sớm, cụ thể, có gợi ý hành động.
* Tôn trọng mọi thành viên trong team, dù là junior hay senior.

---

> “Không cần giỏi ngay từ đầu – chỉ cần tiến bộ mỗi ngày cùng team 💪”

Chào mừng bạn đến với hành trình phát triển một hệ sinh thái giáo dục số mạnh mẽ! 🧡
