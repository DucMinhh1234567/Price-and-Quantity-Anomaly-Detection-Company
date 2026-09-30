# Anomaly Detection — Xuất nhập khẩu

Dự án học máy phát hiện bất thường trên chuỗi thời gian xuất/nhập khẩu (lượng, trị giá, đơn giá), kèm tài liệu bài học và báo cáo EDA.

## Cấu trúc thư mục

```
.
├── data/                 # CSV theo chiều nhap_khau / xuat_khau
├── docs/                 # Ghi chú phân tích theo mặt hàng
├── lessons/              # Tài liệu & notebook theo tuần
├── reports/              # Báo cáo HTML / diagnostics / EDA
├── scripts/              # Notebook & script thử nghiệm
├── requirements.txt
└── README.md
```

## Cài đặt

```bash
python -m venv .venv

# Windows PowerShell
.\.venv\Scripts\Activate.ps1

# macOS / Linux
# source .venv/bin/activate

pip install -r requirements.txt
```

## Phụ thuộc chính

- `pandas`, `numpy`, `scikit-learn`
- `matplotlib`, `seaborn`
- `ydata-profiling`, `pyarrow`, `minio`

## Ghi chú dữ liệu

- Dữ liệu CSV nhỏ trong `data/csv/` được commit cùng repo.
- File `scripts/g_for_g/creditcard.csv` (~144 MB) **không** đưa lên GitHub (vượt giới hạn 100 MB). Tự tải/đặt lại local nếu cần chạy notebook Isolation Forest.

---

## Đưa repo lên GitHub (tự chạy)

Repo hiện **chưa** có `.git`. Chạy lần lượt trong PowerShell tại thư mục project:

### 1. Tạo repo trống trên GitHub

- Vào https://github.com/new
- Đặt tên repo (ví dụ `anomaly-detection`)
- **Không** chọn Add README / .gitignore / license (đã có sẵn local)
- Create repository

### 2. Khởi tạo Git và push

```powershell
cd "C:\Homework\Code File\ML docs\Anomaly Detection"

git init
git add .
git status
git commit -m "Initial commit: anomaly detection project"

git branch -M main
git remote add origin https://github.com/<USERNAME>/<REPO>.git
git push -u origin main
```

Thay `<USERNAME>` và `<REPO>` bằng tài khoản và tên repo của bạn.

### 3. (Tuỳ chọn) Dùng GitHub CLI

Nếu đã cài [GitHub CLI](https://cli.github.com/) và đăng nhập (`gh auth login`):

```powershell
cd "C:\Homework\Code File\ML docs\Anomaly Detection"

git init
git add .
git commit -m "Initial commit: anomaly detection project"
git branch -M main

gh repo create anomaly-detection --private --source=. --remote=origin --push
```

Đổi `--private` thành `--public` nếu muốn repo công khai.

### Xác minh sau khi push

```powershell
git status
git remote -v
```

Trình duyệt mở: `https://github.com/<USERNAME>/<REPO>`
