# 🔚 OFFBOARDING Checklist – API Gateway (DX VAS)

Cảm ơn bạn vì đã đồng hành cùng dự án **API Gateway – Chuyển đổi số VAS**! 🎉

Tài liệu này giúp đảm bảo việc bàn giao, thu hồi quyền truy cập, và lưu giữ tri thức được thực hiện minh bạch, đầy đủ khi một thành viên rời team (tạm thời hoặc vĩnh viễn).

---

## ✅ Checklist khi offboarding

### 1. Thu hồi quyền truy cập

* [ ] Xóa khỏi GitHub organization hoặc repo (`vas-org/api-gateway`)
* [ ] Thu hồi quyền truy cập Google Cloud IAM (Console, Cloud SQL, Artifact Registry)
* [ ] Gỡ khỏi Slack, Notion hoặc tài khoản quản trị nội bộ khác
* [ ] Gỡ quyền khỏi các dịch vụ CI/CD (GitHub Secrets, Deployment Keys, v.v.)

### 2. Bàn giao code và công việc đang dở

* [ ] Toàn bộ nhánh `feature/*` chưa merge đã được:

  * [ ] Tạo Pull Request (nếu đủ điều kiện)
  * [ ] Gán reviewer/assignee thay thế
  * [ ] Ghi chú lại TODO rõ ràng nếu chưa xong
* [ ] Bàn giao issue đã assign trong GitHub Project
* [ ] Nếu có test/manual data/script local, cần push hoặc backup

### 3. Bàn giao tri thức & tài liệu hóa

* [ ] Cập nhật các phần mình phụ trách trong:

  * [ ] `DEV_GUIDE.md`
  * [ ] `CONTRIBUTING.md` (nếu có ảnh hưởng)
  * [ ] Các file `README.md` hoặc `docs/*` của module
* [ ] Nếu có kiến thức riêng (cách setup debug, trick với 1 service...), ghi lại trong `docs/OFFBOARDING_notes/<tên>.md`

### 4. Đánh giá & lời chào

* [ ] Gửi lời cảm ơn qua Slack hoặc email nội bộ
* [ ] Hoàn thành quick retro:

  * [ ] Những gì bạn thấy hiệu quả
  * [ ] Điều gì có thể cải thiện
  * [ ] Bạn muốn để lại lời nhắn gì cho team 🙂

---

## 🧾 Biểu mẫu offboarding mẫu

```md
tên: Nguyễn Văn A
dự án tham gia: API Gateway (DX VAS)
thời gian: Tháng 3/2024 – Tháng 8/2025
bàn giao cho: Trần Thị B (DevOps), Lê Văn C (backend)
retro:
- ✅ Điều hiệu quả: CI/CD nhanh gọn, codebase rõ ràng
- 💡 Nên cải thiện: Rút ngắn thời gian review PR
- ❤️ Lời nhắn: Cảm ơn anh em team, chúc dự án thành công lớn!
```

---

## 📎 Tài liệu hỗ trợ

* Chính sách bảo mật nội bộ (liên hệ IT team)
* Hướng dẫn backup dữ liệu từ máy cá nhân (tuỳ theo công cụ team sử dụng)
* Template: `docs/OFFBOARDING_notes/template.md`

---

> “Bạn có thể rời dự án, nhưng những dòng code và tinh thần đóng góp sẽ còn mãi.” 💛
