---
id: adr-022-sla-slo-monitoring
title: ADR-022: Chiến lược SLA, SLO và Monitoring cho hệ thống dx_vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [sla, slo, monitoring, observability, reliability, dx_vas]
---

## 📌 Bối cảnh

Hệ thống **dx_vas** phục vụ nhiều nhóm người dùng: học sinh, phụ huynh, giáo viên, nhân viên nhà trường. Một số dịch vụ là **business-critical** như API Gateway, LMS Adapter, Frontend SSR. Để đảm bảo chất lượng dịch vụ, cần:
- Định nghĩa rõ ràng SLA/SLO cho từng nhóm dịch vụ
- Giám sát liên tục dựa trên SLI thực tế
- Có alert nếu vượt error budget hoặc vi phạm ngưỡng SLO

---

## 🧠 Quyết định

**Thiết lập khung SLA, SLO, SLI và cơ chế giám sát tuân thủ cho toàn bộ hệ thống dx_vas, với việc phân nhóm dịch vụ, định nghĩa mục tiêu đo lường và tích hợp cảnh báo theo SLO.**

---

## 🧩 Định nghĩa

- **SLA**: Cam kết chính thức (cho stakeholder, người dùng cuối)
- **SLO**: Mục tiêu nội bộ để đo lường độ tin cậy
- **SLI**: Metric dùng để tính toán SLO

---

## 📊 Phân nhóm dịch vụ & SLO đề xuất

| Dịch vụ | Mức độ | SLO uptime | SLI latency (p95) | Error rate max |
|--------|---------|------------|-------------------|----------------|
| API Gateway | Critical | 99.9% (>= 43.2 mins/month downtime allowed) | < 500ms | < 0.1% |
| LMS Adapter | High | 99.5% (<= 3.6h downtime/month) | < 800ms | < 0.5% |
| Frontend SSR | Critical | 99.9% | < 600ms | < 0.2% |
| Notification | Medium | 99% | < 1s | < 1% |
| Sandbox API | Low | 95% | < 2s | < 5% |

> Ghi chú: SLO uptime được tính theo thời gian hoạt động không lỗi chia cho tổng thời gian trong tháng. Ví dụ: 99.9% uptime tương đương tối đa 43.2 phút downtime mỗi tháng (tức 0.1% của 30 ngày).

---

## 📏 Metric đo lường chính (SLI)

| Metric | Định nghĩa |
|--------|------------|
| `availability` | % request thành công / tổng request (dựa trên 2xx + 3xx) |
| `latency_p95` | Thời gian phản hồi của 95% request |
| `error_rate_5xx` | Tỷ lệ request trả về 5xx (server error) |
| `error_rate_4xx` | Tỷ lệ request trả về 4xx không hợp lệ (không tính authentication fail hợp lệ) |
| `uptime` | Trạng thái revision Healthy (Cloud Run), đo mỗi phút |

> **Sự khác biệt giữa uptime và availability:**
> - `uptime`: đo ở mức infrastructure (ví dụ: container healthy, Cloud Run trả 200 tại /healthz)
> - `availability`: đo ở mức ứng dụng (người dùng nhận được response đúng và thành công)
> → cả hai nên được dùng kết hợp: uptime để phát hiện sự cố hệ thống, availability để phản ánh trải nghiệm người dùng.

---

## 🚨 Monitoring & Alerting

- Dùng **Cloud Monitoring SLO dashboard**
- Đặt alert khi:
  - `error_rate_5xx` vượt ngưỡng 3 lần trong 5 phút
  - `latency_p95` > ngưỡng trong 10 phút liên tục
  - `availability` < SLO trong rolling window 30 phút

- Tích hợp alert:
  - Slack, Email, PagerDuty (qua webhook)
  - Đẩy sự kiện vào BigQuery để audit/review sau

---

## 💼 Chính sách phản hồi & đánh giá

- Nếu vượt error budget:
  - Cần freeze release trong vòng 24h
  - Làm postmortem nếu ảnh hưởng user thực > 1 phút downtime hoặc > 100 lỗi

- Mỗi quý review lại:
  - SLO có thực tế và đủ tham vọng chưa?
  - Có cần phân tách SLO theo `env=staging`, `user_type`, `endpoint group` không?
  - Có cần **xây dựng SLO riêng cho môi trường `dev` và `prod`**, trong đó:
    - `prod`: khắt khe hơn, phục vụ người dùng thực
    - `dev/staging`: dùng để phát hiện lỗi sớm, không cảnh báo production

---

## ✅ Lợi ích

- Định hướng rõ ràng cho việc vận hành theo độ tin cậy
- Gắn reliability với việc ra quyết định release
- Phát hiện sớm sự cố trước khi người dùng bị ảnh hưởng lớn

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Alert sai nhiều, gây mù nhiễu | Thêm deadman switch, silence thông minh |
| Thiếu SLI cho 1 số service | Dùng proxy (gateway) để đo từ outside-in |
| SLO quá chặt gây cản release | Định kỳ đánh giá, tách SLO giữa dev & prod |

---

## 📎 Tài liệu liên quan

- Observability: [ADR-005](./adr-005-observability.md)
- External Observability: [ADR-021](./adr-021-external-observability.md)
- CI/CD: [ADR-001](./adr-001-ci-cd.md)
- Auto Scaling: [ADR-016](./adr-016-auto-scaling.md)

---
> “SLA là lời hứa. SLO là la bàn. SLI là chiếc gương sự thật.”
