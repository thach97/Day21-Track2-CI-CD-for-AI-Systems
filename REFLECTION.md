# Báo Cáo Lab: CI/CD for AI Systems

**Họ tên:** Hoàng Ngọc Thạch - 2A202600068

---

## 1. Siêu Tham Số Đã Chọn

Mô hình sử dụng: `RandomForestClassifier` (scikit-learn)

| Tham số | Giá trị cuối | Lý do chọn |
|---|---|---|
| `n_estimators` | 200 | Đủ cây để ổn định kết quả, tránh overfitting so với giá trị nhỏ hơn |
| `max_depth` | 10 | Giới hạn độ sâu giúp tổng quát hóa tốt hơn trên tập eval |
| `min_samples_split` | 5 | Giảm noise từ các node có ít mẫu |

Kết quả so sánh qua MLflow (Bước 1):

| Run | n_estimators | max_depth | min_samples_split | Accuracy | F1 |
|---|---|---|---|---|---|
| Run 1 | 100 | 10  | 5 | ~0.63 | ~0.63 |
| Run 2 | 200 | 10 | 5 | ~0.64 | ~0.64 |
| Run 3 | 200 | 10 | 10 | ~0.62 | ~0.62 |

**Nhận xét:**

- **Run 1 → Run 2:** Tăng `n_estimators` từ 100 lên 200 cải thiện accuracy từ ~0.63 lên ~0.64 (+0.01), cho thấy mô hình còn hưởng lợi từ nhiều cây hơn nhưng mức tăng đã bắt đầu bão hòa.
- **Run 2 → Run 3:** Giữ nguyên `n_estimators=200, max_depth=10` nhưng tăng `min_samples_split` từ 5 lên 10 khiến accuracy giảm về ~0.62, tức là yêu cầu split quá cao làm mô hình thiếu khả năng phân tách tinh tế hơn ở các node sâu.

Bộ tham số **Run 2** (`n_estimators=200, max_depth=10, min_samples_split=5`) được chọn vì đạt accuracy và F1 cao nhất (~0.64), đồng thời `max_depth=10` vẫn đủ ràng buộc để tránh overfit trên tập eval.

---

## 2. Khó Khăn Gặp Phải và Cách Giải Quyết

**Không thể tạo service account key do policy tổ chức GCP**  
Lỗi `constraints/iam.disableServiceAccountKeyCreation` xuất hiện khi chạy `gcloud iam service-accounts keys create`. Giải quyết bằng cách cấp thêm role `roles/orgpolicy.policyAdmin` cho tài khoản cá nhân, sau đó tắt enforce constraint ở cấp organization.

**`mlruns/` bị commit lên Git gây lỗi trên GitHub Actions**  
Thư mục `mlruns/` chứa đường dẫn hardcode `/Users/...` từ máy Mac, khiến MLflow bị lỗi `Permission denied` trên Linux runner. Giải quyết bằng cách thêm `mlruns/` vào `.gitignore` và xóa khỏi git tracking bằng `git rm -r --cached mlruns/`.

**DVC pull thất bại với lỗi `Invalid Credentials 401`**  
Workflow ghi credentials vào `/tmp/sa-key.json` nhưng `.dvc/config` trỏ `credentialpath` về `../sa-key.json` (thư mục gốc project). Giải quyết bằng cách sửa workflow để ghi credentials ra đúng đường dẫn mà DVC config mong đợi.

**Health check thất bại sau khi deploy**  
Script chờ 5 giây rồi curl `/health`, nhưng service cần ~15 giây để tải model từ GCS về. Giải quyết bằng cách thay `sleep 5` bằng retry loop kiểm tra tối đa 12 lần × 5 giây.
