---
id: adr-014-zero-downtime
title: ADR-014 - Chiến lược Triển khai Không Gián Đoạn (Zero-Downtime Deployment) cho hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-05-22
tags: [deployment, cloud-run, traffic-split, dx-vas]
---

## 📌 Bối cảnh

Hệ thống **dx-vas** bao gồm nhiều loại dịch vụ:
- API Gateway (critical)
- Các backend adapters (LMS, CRM, Notification)
- Frontend WebApp (Admin, Portal)

Các dịch vụ này được triển khai trên nền tảng **Google Cloud Run**, vốn hỗ trợ rolling update và traffic splitting qua revision. Tuy nhiên, nếu triển khai không đúng cách, vẫn có thể gây:
- Mất phiên làm việc (session/token invalid)
- Lỗi bất tương thích schema khi database hoặc contract không backward-compatible
- Trạng thái tạm thời "nửa version này, nửa version khác" gây lỗi frontend/backend

## 🧠 Quyết định

**Áp dụng chiến lược triển khai không gián đoạn (zero-downtime) cho toàn hệ thống dx-vas, bằng cách sử dụng Cloud Run revision, kiểm soát traffic shifting, đảm bảo backward compatibility và kiểm thử kỹ trước khi full rollout.**

---

## 🧩 Nguyên tắc triển khai an toàn

### 1. Sử dụng Revision trong Cloud Run
- Mỗi deploy tạo ra 1 revision mới → không ghi đè
- Revisions được định danh theo Git SHA hoặc `app_version`

### 2. Traffic Splitting / Canary
- Bắt đầu với `1%`, `10%`, `25%`, `50%` → kiểm tra log, error rate, latency
- Sau đó mới `100%` → stable rollout
- Có thể rollback ngay về revision cũ nếu lỗi xuất hiện

### 3. Backward Compatibility (BC)
- Mọi API mới phải:
  - Giữ nguyên format response cũ
  - Không bắt buộc các field mới khi deserialize (frontend, consumer)
  - Đảm bảo DB migration là additive (`ADD COLUMN`, không `DROP`/`RENAME` ngay)

### 4. DB Migration an toàn
- Migration tách biệt khỏi deploy (chạy trước)
- Add field trước – app dùng field sau
- Sau khi deploy ổn định → cleanup field cũ (qua release sau)

### 5. Health Check & Startup Readiness
- Sử dụng `startup probe` hoặc `readiness probe` để tránh nhận traffic quá sớm
- Bắt buộc trả HTTP 200 tại `/healthz`, `/readyz` trước khi nhận request thật

### 6. Session/Token Stability
- JWT ký với key chưa thay đổi → client không bị logout
- Không xóa refresh token / session store trong quá trình deploy

---

## 🧪 Tích hợp CI/CD
- Triển khai staging auto 100% traffic khi merge vào `dev`
- Triển khai production manual + canary khi merge vào `main`
- Canary theo tag: `--tag canary-v1`
- Rollback qua UI hoặc `gcloud run services update-traffic`

---

## 🔧 Áp dụng linh hoạt theo dịch vụ

| Loại service | Chiến lược cụ thể |
|--------------|-------------------|
| API Gateway | Canary + rollout chặt, BC nghiêm ngặt |
| LMS/CRM Adapter | Có thể rollout nhanh hơn, miễn không vi phạm contract |
| Frontend Webapp | Triển khai SSR Cloud Run → nên dùng Canary nếu gắn session/token |
| Notification Service | Ít critical hơn → rollout nhanh, test kỹ queue retry |

---

## ✅ Lợi ích

- Tránh downtime gây gián đoạn cho người dùng
- Triển khai có thể rollback nhanh chóng
- Đảm bảo ổn định hệ thống dù đang nâng cấp

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Đẩy traffic quá nhanh → không phát hiện lỗi kịp | Chia nhỏ canary, theo dõi error rate, latency |
| Bất tương thích DB → lỗi runtime | Luôn chạy migration tách biệt, forward-compatible trước |
| Version mismatch giữa frontend/backend | Dùng header `X-App-Version`, log mismatch, tránh breaking change |

---

## 🔄 Các phương án đã loại bỏ

| Phương án | Lý do không chọn |
|-----------|------------------|
| Redeploy đè revision (default) | Không rollback được, dễ mất session, khó trace lỗi |
| Manual rollout không kiểm soát traffic | Không phù hợp hệ thống cần SLA cao |

---

## 📎 Tài liệu liên quan

- API Governance: [ADR-009](./adr-009-api-governance.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)
- Env Config: [ADR-005](./adr-005-env-config.md)

---
> “Triển khai không gián đoạn không phải là may mắn – mà là kỹ thuật.”
