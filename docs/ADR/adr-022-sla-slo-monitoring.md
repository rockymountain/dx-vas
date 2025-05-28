---
id: adr-022-sla-slo-monitoring
title: ADR-022 - Chiến lược SLA, SLO và Monitoring cho hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-05-22
tags: [sla, slo, monitoring, observability, reliability, dx-vas]
---

# ADR-022: SLA, SLO và Giám sát (Monitoring Strategy)

## Trạng thái

Accepted

## Bối cảnh

Hệ thống dx-vas phục vụ nhiều trường thành viên (multi-tenant). Để đảm bảo chất lượng dịch vụ và dễ truy vết khi có sự cố, hệ thống cần định nghĩa rõ SLA/SLO cho từng thành phần và có hệ thống giám sát phù hợp.

Mỗi tenant có stack dịch vụ riêng biệt (Sub Service, Frontend, Adapter...), do đó việc theo dõi riêng theo từng tenant là cần thiết.

## Quyết định

### 1. Định nghĩa SLA/SLO cho từng nhóm dịch vụ

| Dịch vụ | SLO đề xuất | SLA target |
|--------|-------------|------------|
| Auth (Master/Sub) | 99.9% request thành công dưới 500ms | ≥ 99.5% |
| API Gateway | 99.9% routing chính xác, không lỗi 5xx | ≥ 99.9% |
| Sub User / Sub Notification | 99.9% uptime / latency ổn định < 700ms | ≥ 99.5% |
| Pub/Sub Processing | ≤ 5s từ lúc publish → xử lý xong tại Sub Service | ≥ 99.0% |

### 2. Mỗi tenant stack có metric & alert riêng

- Mỗi tenant có dashboard riêng (dùng Cloud Monitoring hoặc Grafana)
- Mỗi alert phải gắn label `tenant_id`, ví dụ:

```yaml
labels:
  tenant_id: "abc"
  service: "sub-user"
  severity: "critical"
```

* Superadmin có giao diện tổng hợp alert theo tenant
* Các chỉ số được phân loại theo nhóm:

| Nhóm         | Ví dụ chỉ số                                          |
| ------------ | ----------------------------------------------------- |
| Latency      | P95 response time per service per tenant              |
| Availability | Uptime Sub Service theo `tenant_id`                   |
| Error Rate   | 5xx ratio, failed login OTP, message delivery failure |
| Pub/Sub      | Event delay, processing time, ack timeout per tenant  |

### 3. Log tập trung theo tenant

* Tất cả log đều có field `tenant_id`
* Cho phép truy vấn nhanh theo tenant:

```sql
SELECT * FROM logs
WHERE tenant_id = "abc"
AND severity = "ERROR"
AND timestamp > TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 1 HOUR)
```

* Logs gửi về Stackdriver hoặc BigQuery để phân tích

### 4. Alert routing và escalation

* Alert gửi qua Zalo/Email theo tenant
* Alert `core service` gửi về nhóm DevOps chính
* Alert `tenant stack` có thể gửi riêng cho quản trị viên kỹ thuật của trường

## Hệ quả

✅ Ưu điểm:

* Dễ truy vết lỗi theo từng trường/tenant
* Giúp kiểm soát chất lượng dịch vụ đa tầng (core + tenant)
* Hỗ trợ tính toán SLA/SLO minh bạch với khách hàng (trường)

⚠️ Lưu ý:

* Cần công cụ tự động cấu hình alert theo tenant mới
* Phải đảm bảo log không bị trộn giữa tenants

## Liên kết liên quan

* [`adr-015-deployment-strategy.md`](./adr-015-deployment-strategy.md)
* [`adr-004-security.md`](./adr-004-security.md)
* [`adr-008-audit-logging.md`](./adr-008-audit-logging.md)

---
> “SLA là lời hứa. SLO là la bàn. SLI là chiếc gương sự thật.”
