---
id: adr-016-auto-scaling
title: ADR-016: Chiến lược Auto Scaling cho hệ thống dx-vas
status: accepted
author: DX VAS Platform Team
date: 2025-06-22
tags: [scaling, cloud-run, performance, cost, dx-vas]
---

## 📌 Bối cảnh

Các dịch vụ trong hệ thống **dx-vas** (API Gateway, LMS Adapter, CRM Proxy, SSR frontend...) đều triển khai trên **Google Cloud Run**, nơi mỗi service có thể scale độc lập. Việc không tối ưu hóa auto scaling sẽ dẫn đến:
- Chậm phản hồi hoặc timeout khi traffic tăng cao đột ngột
- Tốn chi phí khi duy trì quá nhiều instance không cần thiết
- Dễ vượt quota hoặc bị rate-limit nếu concurrency không được cấu hình đúng

## 🧠 Quyết định

**Áp dụng chiến lược auto scaling tối ưu cho từng loại service trong hệ thống dx-vas, sử dụng Cloud Run concurrency, min/max instance và metric-based scaling để cân bằng chi phí – hiệu năng – độ tin cậy.**

---

## ⚙️ Cấu hình Cloud Run đề xuất

| Loại service | Concurrency | Min instance | Max instance | Khác |
|--------------|-------------|--------------|--------------|-------|
| API Gateway | 1 | 2 (always warm) | 10 | latency-sensitive |
| LMS/CRM Adapter | 10 | 0 | 20 | xử lý IO/batch tốt |
| Frontend SSR | 2 | 1 | 5 | cần warm start để SSR nhanh |
| Notification Service | 20 | 0 | 5 | background burst |

### ✅ Giải thích:
- **Concurrency**:
  - API Gateway được cấu hình `concurrency = 1` do đặc tính **latency-sensitive** và để đảm bảo **mỗi request được xử lý nhanh nhất có thể trên một instance**, tránh ảnh hưởng lẫn nhau giữa các request song song. Điều này giúp giảm độ trễ từng request, nhưng **có thể dẫn đến cần nhiều instance hơn khi traffic cao**. Giá trị này **sẽ được theo dõi và tinh chỉnh dựa trên kết quả load testing và dữ liệu vận hành thực tế**.
  - `10–20` cho service IO-bound (adapter, notification)

- **Min instance**:
  - `>0` nếu muốn warm start (frontend SSR, Gateway)
  - `0` cho service không cần sẵn sàng liên tục (adapter, batch)

- **Max instance**:
  - Tùy vào quota, chi phí và SLA

---

## 🔍 Auto Scaling theo Metric

Cloud Run scale dựa trên:
- CPU utilization
- Request latency / concurrency
- Traffic volume (HTTP request rate)

> Ghi chú: hành vi "scale out nếu error rate tăng cao + concurrency > ngưỡng" là **quan sát** từ behavior mặc định của Cloud Run, chứ không phải một hành động chủ động tự động hoá. Tuy nhiên, các alert này **có thể được dùng để trigger hành động như tăng `min_instances` tạm thời hoặc kiểm tra lại cấu hình autoscaling.**

Sẽ tích hợp alert qua:
- Cloud Monitoring: alert nếu request latency > 2s
- Scale out behavior được hỗ trợ tự động, nhưng có thể được kết hợp với script điều chỉnh thủ công nếu cần

---

## 🛠 Tích hợp CI/CD & Terraform

- Mỗi service có file `infra.tf` định nghĩa:
```hcl
resource "google_cloud_run_service" "service_name" {
  template {
    spec {
      container_concurrency = 10
      min_instances = 0
      max_instances = 20
    }
  }
}
```
- GitHub Actions sẽ validate scaling config trước deploy (check hard limit, quota)
- Có thể tách autoscaling policy ra file riêng (module reusable)

---

## ✅ Lợi ích

- Tăng hiệu năng khi cần, tiết kiệm khi idle
- Tránh cold start với dịch vụ cần phản hồi nhanh (gateway, frontend SSR)
- Chủ động kiểm soát chi phí Cloud Run

## ❌ Rủi ro & Giải pháp

| Rủi ro | Giải pháp |
|--------|-----------|
| Scale quá nhanh → backend overload | Giới hạn `max_instance` + alert precondition |
| Cold start gây chậm cho user đầu tiên | Giữ `min_instance > 0` cho service critical |
| Thay đổi concurrency không đồng bộ với logic service | Document rõ ràng, enforced qua CI review và thông qua **load testing** kỹ lưỡng |

---

## 📎 Tài liệu liên quan

- CI/CD Strategy: [ADR-001](./adr-001-ci-cd.md)
- Zero downtime: [ADR-014](./adr-014-zero-downtime.md)
- Deployment Strategy: [ADR-015](./adr-015-deployment-strategy.md)

---
> “Auto Scaling không chỉ để tiết kiệm – mà là để đảm bảo bạn luôn sẵn sàng.”
