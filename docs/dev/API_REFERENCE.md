# 📘 API Reference – API Gateway (DX VAS)

Tài liệu này cung cấp cái nhìn tổng quan về các endpoint chính trong hệ thống **API Gateway** của dự án **Chuyển đổi số VAS**.

> 💡 Tài liệu này được tổng hợp thủ công từ các route chính. Với các thông tin chi tiết như schema đầu vào/đầu ra, vui lòng tham khảo Swagger UI tại `/docs` hoặc ReDoc tại `/redoc`.

---

## ✅ Chuẩn REST & Headers chung

* **Prefix version:** `/api/v1/`
* **Headers bắt buộc:**

  * `Authorization: Bearer <access_token>`
  * Forward đến backend sẽ có thêm:

    * `X-User-Id`
    * `X-Role`
    * `X-Permissions`
* **Response chuẩn hóa:**

```json
{
  "data": {...},
  "message": "Success",
  "status_code": 200
}
```

* **Mã lỗi phổ biến:**

  * `401 Unauthorized` – Thiếu hoặc sai token
  * `403 Forbidden` – Không đủ quyền truy cập
  * `429 Too Many Requests` – Vượt giới hạn
  * `502 Bad Gateway` – Lỗi từ backend

---

## 🔐 Auth (OAuth2, Token)

### `POST /auth/login`

* Mô tả: Đăng nhập bằng Google OAuth2, trả về token và thông tin user
* Đầu vào:

```json
{
  "code": "google_oauth_code"
}
```

* Đầu ra (ví dụ):

```json
{
  "data": {
    "access_token": "jwt",
    "refresh_token": "string",
    "token_type": "bearer",
    "user_info": {
      "id": "string",
      "email": "string",
      "name": "string"
    }
  },
  "message": "Login successful",
  "status_code": 200
}
```

### `POST /auth/refresh`

* Mô tả: Cấp lại Access Token từ Refresh Token
* Đầu vào:

```json
{
  "refresh_token": "string"
}
```

### `GET /auth/me`

* Mô tả: Lấy thông tin người dùng hiện tại dựa trên token đã đăng nhập

---

## 👤 RBAC – Role & Permission

### `GET /rbac/roles`

* Mô tả: Lấy danh sách role đang có trong hệ thống

### `POST /rbac/roles`

* Mô tả: Tạo role mới
* Yêu cầu quyền: `MANAGE_RBAC`

### `GET /rbac/permissions`

* Mô tả: Lấy danh sách permission hiện tại (code + mô tả)

### `POST /rbac/assign`

* Mô tả: Gán role cho user
* Đầu vào:

```json
{
  "user_id": "string",
  "role_id": "string"
}
```

### `POST /rbac/route-map`

* Mô tả: Gán permission cụ thể cho endpoint (method + path)

> 🔎 Xem mô tả các permission tại [`DEV_GUIDE.md`](./DEV_GUIDE.md)

---

## 🎓 SIS (Proxy đến OpenSIS)

### `GET /sis/students`

* Mô tả: Lấy danh sách học sinh
* Yêu cầu quyền: `VIEW_STUDENT`
* Query params:

  * `page`, `limit`, `search`

### `GET /sis/students/{id}`

* Mô tả: Xem chi tiết một học sinh
* Yêu cầu quyền: `VIEW_STUDENT`

### `PUT /sis/students/{id}`

* Mô tả: Cập nhật thông tin học sinh
* Yêu cầu quyền: `EDIT_STUDENT`

---

## 📞 Notify

### `POST /notify/send`

* Mô tả: Gửi thông báo đến user qua Web, Zalo hoặc Email
* Đầu vào ví dụ:

```json
{
  "user_id": "abc123",
  "channel": "email",
  "message": "Bạn có thông báo mới."
}
```

---

## 🧑‍💼 CRM (Proxy đến EspoCRM – nếu áp dụng)

### `GET /crm/leads`

* Mô tả: Lấy danh sách phụ huynh tiềm năng

### `POST /crm/leads`

* Mô tả: Thêm mới thông tin đăng ký nhập học

---

## 🧑‍🏫 LMS (Proxy đến Moodle – nếu đã tích hợp)

### `GET /lms/courses`

* Mô tả: Lấy danh sách khoá học từ LMS

### `GET /lms/students/{id}/grades`

* Mô tả: Lấy điểm học tập của học sinh từ Moodle

---

## 🔗 Tài nguyên liên quan

* Swagger UI: [`/docs`](http://localhost:8000/docs)
* ReDoc UI: [`/redoc`](http://localhost:8000/redoc)
* Quy ước coding: [`CONTRIBUTING.md`](./CONTRIBUTING.md)
* Dev Guide: [`DEV_GUIDE.md`](./DEV_GUIDE.md)

---

> Tài liệu này là bản mô tả thủ công (version: `v1`). Để cập nhật đầy đủ nhất, hãy tham khảo Swagger UI.
