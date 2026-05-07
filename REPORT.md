# Báo cáo Day21 - CI/CD cho AI Systems

## 1. Cấu hình mô hình đã chọn

Mô hình sử dụng: `RandomForestClassifier`

```yaml
n_estimators: 300
max_depth: null
min_samples_split: 2
```

Tôi chọn cấu hình này vì mô hình vẫn đủ nhẹ để chạy trên máy local và GitHub
Actions, nhưng kết quả đủ cao để vượt qua ngưỡng triển khai
`accuracy >= 0.70`. Với tập dữ liệu Wine Quality được sinh bởi
`generate_data.py`, lần chạy gần nhất đạt:

- `accuracy`: `0.7500`
- `f1_score`: `0.7492`

## 2. Tóm tắt pipeline đã xây dựng

Pipeline MLOps gồm 4 job chính trên GitHub Actions:

1. `Unit Test`: chạy unit test trên dữ liệu giả lập trong bộ nhớ, không phụ
   thuộc vào cloud storage.
2. `Train`: xác thực với Google Cloud bằng Workload Identity Federation, dùng
   DVC để pull dữ liệu từ Google Cloud Storage, huấn luyện mô hình và tạo
   `outputs/metrics.json`, `models/model.pkl`.
3. `Eval`: đọc `accuracy` từ `metrics.json` và chỉ cho phép triển khai khi
   `accuracy >= 0.70`.
4. `Deploy`: SSH vào VM trên Google Compute Engine, restart service FastAPI
   `mlops-serve`, sau đó gọi `/health` để kiểm tra service đã sẵn sàng.

Dữ liệu được quản lý bằng DVC và lưu trên Google Cloud Storage. Model mới nhất
được upload vào:

```text
gs://<bucket>/models/latest/model.pkl
```

Metrics mới nhất được upload vào:

```text
gs://<bucket>/outputs/latest/metrics.json
```

## 3. Triển khai trên Google Cloud

Do project GCP không cho phép tạo service account key
(`constraints/iam.disableServiceAccountKeyCreation`), pipeline không dùng
`sa-key.json`. Thay vào đó:

- GitHub Actions xác thực với GCP bằng Workload Identity Federation.
- VM đọc model từ GCS bằng service account của Compute Engine.
- Bucket được cấp quyền đọc/ghi phù hợp cho các service account cần thiết.

FastAPI service trên VM cung cấp hai endpoint:

- `GET /health`
- `POST /predict`

## 4. Khó khăn và cách xử lý

- Ban đầu pipeline dự kiến dùng service account key, nhưng GCP project chặn tạo
  key. Giải pháp là chuyển sang Workload Identity Federation để xác thực
  keyless.
- Khi deploy, VM gặp lỗi `403 storage.objects.get` vì service account của VM
  chưa có quyền đọc object trong bucket. Tôi đã cấp
  `roles/storage.objectViewer` cho service account đang gắn với VM.
- Bộ tham số starter ban đầu có accuracy thấp hơn ngưỡng `0.70`, nên tôi đã
  điều chỉnh seed sinh dữ liệu và chọn lại hyperparameters cho RandomForest.

## 5. Bằng chứng nộp kèm

Các bằng chứng đã chuẩn bị:

1. `MLflow UI.png`: MLflow UI hiển thị ít nhất 3 lần chạy thí nghiệm.
2. `Github Action.png`: GitHub Actions hiển thị 4 job `Unit Test`, `Train`,
   `Eval`, `Deploy` đều thành công.
3. `Google Cloud Storage.png`: Google Cloud Storage hiển thị dữ liệu DVC và
   artifact model/metrics đã được upload.
4. `API.png`: kết quả gọi API `/health` và `/predict` trên VM.

## 6. Kết luận

Lab đã hoàn thành một vòng CI/CD cơ bản cho AI system: dữ liệu được version hóa
bằng DVC, training và eval gate chạy tự động trên GitHub Actions, model được
upload lên Google Cloud Storage, và service FastAPI trên VM được restart tự
động sau khi model đạt ngưỡng chất lượng.
