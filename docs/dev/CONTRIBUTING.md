# 🤝 Hướng dẫn Đóng góp – API Gateway (DX VAS)

Cảm ơn bạn đã quan tâm và muốn đóng góp vào dự án **API Gateway – Chuyển đổi số VAS**! Dưới đây là hướng dẫn giúp bạn tham gia hiệu quả, đảm bảo tiêu chuẩn chất lượng và sự nhất quán trong quy trình phát triển.

---

## 📌 Mục tiêu

* Duy trì chất lượng code cao, sạch và có kiểm thử.
* Đảm bảo mọi thay đổi đều được review, ghi nhận và rollback được.
* Khuyến khích đóng góp từ nội bộ và cộng đồng mở rộng (nếu applicable).

---

## 🧱 Cấu trúc nhánh (Branch Strategy)

| Mục đích           | Nhánh                     |
| ------------------ | ------------------------- |
| Production         | `main`                    |
| Staging / tích hợp | `dev`                     |
| Tính năng mới      | `feature/<ten-tinh-nang>` |
| Hotfix             | `hotfix/<ten-loi>`        |

---

## ⚙️ Quy trình đóng góp

1. **Tạo nhánh mới** từ `dev`:

```bash
git checkout dev
git pull origin dev
git checkout -b feature/ten-tinh-nang
```

2. **Commit theo chuẩn Conventional Commit**:

```bash
git commit -m "feat(auth): thêm xác thực OAuth2 cho giáo viên"
```

3. **Rebase trước khi tạo Pull Request**:

```bash
git fetch origin
git rebase origin/dev
# Nếu nhánh feature đã push từ trước:
git push --force-with-lease
```

4. **Push và tạo Pull Request về `dev`**. PR nên nhỏ, tập trung một chức năng, dễ review.

5. **Chờ review tối thiểu 1 thành viên** (ưu tiên senior dev hoặc tech lead với module phức tạp).

6. **Resolve conflict (nếu có)** và cập nhật lại PR.

7. **Squash & Merge** nếu được chấp thuận.

---

## ✅ Checklist trước khi tạo PR

* [ ] Đã viết test (unit/integration nếu applicable)?
* [ ] Đã chạy `pre-commit` và pass lint?
* [ ] Tài liệu (docstring, README nếu cần) đã cập nhật?
* [ ] Đã chạy `docker-compose up` và verify hoạt động local?
* [ ] PR nhỏ, gọn, tập trung 1 mục tiêu?

---

## ✍️ Coding Convention

* Tuân theo [PEP8](https://peps.python.org/pep-0008/)
* Sử dụng **type hinting đầy đủ**
* Viết docstring theo [Google Style Guide](https://google.github.io/styleguide/pyguide.html)
* Xem thêm chi tiết tại [`docs/DEV_GUIDE.md`](./DEV_GUIDE.md)
* Sử dụng custom exception theo từng domain nếu có logic nghiệp vụ

---

## 🧪 Testing & CI

* Unit test dùng `pytest`
* Tối thiểu 1 test cho mỗi logic quan trọng hoặc edge case
* Coverage mục tiêu ≥ 80%
* CI pipeline tự động chạy:

  * `black`, `flake8`, `isort`
  * `pytest`
  * `bandit`, `safety`, `trivy`

---

## 🐞 Quy trình xử lý Issue/Bug

* Báo cáo lỗi qua GitHub Issues hoặc Slack nội bộ
* Gán label `bug`, `enhancement`, `help wanted`, v.v.
* Chỉ nhận xử lý issue khi đã **assign bản thân** và thông báo với team
* Các issue nghiêm trọng nên được ghi rõ reproduction steps, log, ảnh screenshot nếu cần

---

## 🔍 Review code

* Tối thiểu **1 reviewer** phải approve trước khi merge
* Reviewer nên là **senior dev** hoặc **module owner** cho các thay đổi lớn
* Review cần diễn ra trong vòng **48 giờ**
* Góp ý nên cụ thể, tích cực và rõ ràng
* Tranh luận kỹ thuật nên được trao đổi trong GitHub comment hoặc Slack; escalate nếu cần

---

## 💬 Giao tiếp & Phản hồi

* Sử dụng Slack, GitHub Discussion hoặc Issue tracker
* Tôn trọng thời gian của nhau, tránh push commit cuối ngày rồi yêu cầu review gấp
* Reviewer có thể từ chối PR nếu:

  * Không đủ test
  * Không đúng convention
  * Gây ảnh hưởng đến module khác mà không có phối hợp

---

## 📄 License & Quyền sở hữu

Mọi đóng góp được xem là thuộc về tổ chức **Trường Việt Anh – Dự án Chuyển đổi số VAS**. Việc đóng góp đồng nghĩa với việc bạn đồng thuận với các điều khoản nội bộ đã được thống nhất.

---

> “Mỗi dòng code bạn viết hôm nay là nền tảng cho một ngôi trường số thông minh ngày mai.”

**Chúng tôi rất mong đợi sự đóng góp của bạn! ❤️**
