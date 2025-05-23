---
id: adr-021-external-observability
title: ADR-021: Chính sách tích hợp Observability với hệ thống bên ngoài cho dx_vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [observability, logging, metrics, tracing, dx_vas, external, otlp]
---

## 📌 Bối cảnh

Trong hệ thống **dx_vas**, phần lớn dữ liệu log, metric và trace đang được thu thập và giám sát qua Google Cloud Platform (Cloud Logging, Cloud Monitoring, Cloud Trace). Tuy nhiên, tổ chức có thể có nhu cầu:
- Tập trung hóa dữ liệu vào một hệ thống giám sát hiện có như **Datadog**, **Grafana Cloud**, **ELK**, **Sentry**, hoặc **New Relic**
- Tích hợp dữ liệu cảnh báo với hệ thống **incident management** hoặc **Security Operations Center (SOC)** hiện tại
- Phân tích log/metric theo cách Google Cloud không hỗ trợ tốt

Việc tích hợp external observability này cần được chuẩn hóa và bảo vệ để:
- Đảm bảo bảo mật và tuân thủ dữ liệu
- Kiểm soát chi phí egress traffic và licensing
- Giữ nguyên trace ID và context để phân tích liên hệ giữa hệ thống nội bộ và bên ngoài

---

## 🧠 Quyết định

**Hệ thống dx_vas cho phép và hỗ trợ tích hợp với các hệ thống observability bên ngoài khi có nhu cầu chính đáng, theo các tiêu chuẩn kỹ thuật, bảo mật và chi phí được định nghĩa trong tài liệu này.**

---

## 🔍 Khi nào nên tích hợp với hệ thống bên ngoài?

- Có hệ thống SOC, SIEM hoặc centralized monitoring ngoài GCP
- Cần tính năng mà Cloud Monitoring không đáp ứng được (alert logic phức tạp, correlation, SLO dashboard tuỳ chỉnh...)
- Yêu cầu tuân thủ tổ chức (log phải lưu trữ tập trung)
- Dữ liệu phục vụ cross-system analysis (backend từ nhiều nền tảng)

---

## 📦 Loại dữ liệu được export

| Loại | Ví dụ | Điều kiện |
|------|-------|-----------|
| Logs | error logs, audit logs | Mask nhạy cảm, giữ trace_id |
| Metrics | latency, request count, memory | Có label `dx_vas_service` và `env` |
| Traces | Cloud Trace → OTLP span | Mapping span name, attributes chuẩn |
| Alerts/Events | Deployment alert, billing alert | Webhook hoặc Pub/Sub stream |

---

## 🔗 Giao thức & phương thức tích hợp chuẩn

| Dữ liệu | Giao thức | Phương pháp | Ghi chú |
|--------|-----------|-------------|--------|
| Logs | Pub/Sub, OTLP, HTTPS | Logging Sink → Pub/Sub → forwarder | Forward từ Pub/Sub đến Fluent Bit / Logstash agent |
| Metrics | OTLP, Prometheus remote write | OpenTelemetry Collector → Backend | Expose metrics nếu cần pull |
| Traces | OTLP | OTEL Collector | Hỗ trợ span enrich + mapping |
| Alerts | Webhook | Cloud Monitoring → Webhook | Tới Slack, PagerDuty, external API |

---

## ✅ Các hệ thống external có thể tích hợp

| Hệ thống | Loại | Ghi chú |
|----------|------|--------|
| Datadog | Log, Metric, APM | Có agent + API push |
| Grafana Cloud | Metric, Logs | Qua OTLP hoặc Prometheus remote write |
| Sentry | Error tracking | SDK hoặc forwarding log với context |
| New Relic | Metric, APM | Tích hợp OTLP chuẩn |
| ELK / OpenSearch | Logs | Qua Fluent Bit hoặc Logstash |

### Nguyên tắc lựa chọn hệ thống:
- Hỗ trợ chuẩn mở (OTLP, Prometheus, OpenTracing)
- Có cơ chế bảo vệ, xác thực, giới hạn truy cập
- Tính năng vượt trội so với GCP nếu dùng song song
- Có cảnh báo nếu chi phí vượt dự đoán

---

## 🧾 Tagging & Metadata chuẩn hoá

Mọi dữ liệu gửi ra phải có các metadata sau:
```yaml
trace_id: ...
correlation_id: ...
env: production|staging|dev
service: api_gateway | lms_adapter | ...
module: auth | user | crm
team: platform | lms | frontend
```

---

## 🔐 Bảo mật & tuân thủ

- Mọi log chứa dữ liệu người dùng phải được **mask** trước khi export
- Chỉ export từ môi trường `staging`, `prod`, không từ `dev` nếu không có nhu cầu giám sát
- Dùng HTTPS hoặc mTLS cho webhook/API endpoint
- Đặt rule IAM rõ ràng cho bên nhận dữ liệu
- Đảm bảo region residency nếu có ràng buộc chủ quyền dữ liệu

---

## 💵 Kiểm soát chi phí

| Chi phí | Quản lý |
|---------|---------|
| Egress traffic | Gắn label trace/log để tính theo team |
| External licensing | Tách bill theo project/module |
| Volume logs/traces | Áp dụng sampling hoặc filter logs từ Cloud Logging |

---

## ✅ Lợi ích

- Cho phép đội DevOps/SOC tích hợp giám sát linh hoạt
- Đảm bảo compliance và khả năng điều tra tốt hơn
- Dễ tích hợp với quy trình vận hành/phân tích sự cố của tổ chức

---

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Dữ liệu rò rỉ hoặc export sai | Masking, kiểm tra định kỳ, trace log outbound |
| Egress traffic vượt kiểm soát | Sampling, alert trên volume log |
| Dữ liệu export không liên kết được với trong GCP | Gắn `trace_id`, `correlation_id` đầy đủ |

---

## 📎 Tài liệu liên quan

- Cost Observability: [ADR-020](./adr-020-cost-observability.md)
- Auto Scaling: [ADR-016](./adr-016-auto-scaling.md)
- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)
- Security: [ADR-004](./adr-004-security.md)
- Secrets: [ADR-003](./adr-003-secrets.md)

---
> "Observability không nên bị giới hạn trong nội bộ – nếu việc nhìn xa giúp bạn phản ứng nhanh hơn."
