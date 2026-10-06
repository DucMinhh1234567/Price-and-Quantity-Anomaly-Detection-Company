# Phân tích prototype cảnh báo bất thường — Xuất khẩu gạo

Tài liệu giải thích kỹ thuật trong `rice_alert_prototype.py` / `rice_alert_prototype.ipynb`: mô hình dùng gì, xử lý dữ liệu ra sao, và quy tắc cảnh báo đa chỉ tiêu.

---

## 1. Bài toán

| Mục | Chi tiết |
|---|---|
| Dữ liệu | Chuỗi tháng xuất khẩu gạo (`data/csv/xuat_khau/gạo.csv`) |
| Chỉ tiêu | `so_luong`, `tri_gia`, `don_gia = tri_gia / so_luong` |
| Nhãn | Không có (unsupervised) |
| Mục tiêu | Cảnh báo tháng lệch mạnh so với kỳ vọng dự báo 1 bước |
| Ràng buộc quan trọng | Tại tháng $t$ **không dùng dữ liệu ≥ t** (no look-ahead) |

Đây là baseline kiểu **forecast residual + robust scoring**, phù hợp chuỗi tháng có xu hướng và mùa vụ nhẹ.

---

## 2. Pipeline tổng quan

```
CSV tháng
  → index MS + don_gia
  → log(y)
  → Holt-Winters 1-step (cửa sổ trượt)
  → residual e_t = y_t − ŷ_t
  → median/MAD của e_{≤t−1} → robust z
  → gộp 3 chỉ tiêu → OK / WARN / CRITICAL
```

---

## 3. Tiền xử lý

1. **Parse thời gian**: `thang_nam` → datetime index, sắp xếp tăng dần.
2. **Tần suất đều**: `asfreq("MS")` để đảm bảo chuỗi tháng đầu kỳ liên tục.
3. **Đơn giá**: `don_gia = tri_gia / so_luong` — tách anomaly **giá** khỏi anomaly **khối lượng**.
4. **Thang log**: huấn luyện / scoring trên $\log(\cdot)$ để ổn định phương sai và làm residual gần cộng tính hơn.

Không nội suy missing trong prototype này (chuỗi gạo khá đầy đủ). Với mặt hàng khác có null giữa kỳ, cần cắt lịch sử hoặc mask — không điền 0.

---

## 4. Mô hình dự báo: Holt-Winters (ETS)

### 4.1. Thuật toán

Dùng `statsmodels.tsa.holtwinters.ExponentialSmoothing` với:

| Tham số | Giá trị | Ý nghĩa |
|---|---|---|
| `trend` | `"add"` | Xu hướng cộng tính |
| `damped_trend` | `True` | Xu hướng bị damped (tránh phóng đại dài hạn) |
| `seasonal` | `"add"` | Mùa vụ cộng tính |
| `seasonal_periods` | `12` | Chu kỳ năm (dữ liệu tháng) |

Đây là mô hình **ETS (Error-Trend-Seasonal)** dạng additive damped, thuộc nhóm **forecast residual anomaly detection**.

### 4.2. Cơ chế walk-forward (không nhìn tương lai)

Với mỗi tháng $t$ từ `MIN_TRAIN` trở đi:

1. Lấy lịch sử `y[max(0, t−WINDOW) : t]` — cửa sổ tối đa `WINDOW = 96` tháng.
2. Fit Holt-Winters trên lịch sử đó.
3. Forecast **1 bước** → $\hat{y}_t$.
4. Nếu fit lỗi → fallback: trung bình 12 tháng gần nhất.

| Tham số | Mặc định | Vai trò |
|---|---|---|
| `MIN_TRAIN` | 60 | Số tháng tối thiểu trước khi bắt đầu chấm điểm |
| `WINDOW` | 96 | Cửa sổ trượt tối đa để fit |

### 4.3. Vì sao chọn Holt-Winters?

- Phù hợp chuỗi tháng có **trend + seasonality** nhẹ.
- Fit nhanh, dễ giải thích hơn Prophet / deep learning.
- Residual có ý nghĩa nghiệp vụ: “tháng này lệch bao nhiêu so với kỳ vọng gần đây”.

Hạn chế: giả định mùa vụ ổn định; đột biến cấu trúc dài hạn có thể làm residual “dính” vài tháng sau khi xảy ra.

---

## 5. Scoring: robust z-score trên residual

### 5.1. Residual

$$
e_t = y_t - \hat{y}_t
\quad (y = \log(\text{metric}))
$$

### 5.2. Ước lượng vị trí & thang đo (chỉ dùng quá khứ)

Để tránh look-ahead khi ước lượng phân phối residual:

$$
m_t = \mathrm{median}(e_{t-W_s},\ldots,e_{t-1}),\quad
\mathrm{MAD}_t = \mathrm{median}\big(|e_{t-W_s}-m_t|,\ldots,|e_{t-1}-m_t|\big)
$$

với `SCALE_WIN = 36`, `min_periods = 12`, và dùng `shift(1)` trước khi rolling.

### 5.3. Robust z

$$
z_t = \frac{e_t - m_t}{\mathrm{MAD}_t / 0.6745}
$$

Hệ số **0.6745** đưa MAD về ước lượng tương đương độ lệch chuẩn dưới giả định chuẩn. `scale` được clip tối thiểu `1e-3` để tránh chia cho 0.

**Vì sao MAD thay vì std?** Median/MAD bền vững hơn mean/std khi chính residual đã có outlier — tránh “outlier làm loãng ngưỡng” rồi bỏ sót điểm sau.

---

## 6. Quy tắc cảnh báo đa chỉ tiêu

Ngưỡng từng chỉ tiêu:

| Mức | Điều kiện trên trị tuyệt đối z |
|---|---|
| WARN | $\geq 3.0$ |
| CRITICAL (đơn chỉ tiêu) | $\geq 4.5$ |

Gộp 3 cột `z_so_luong`, `z_tri_gia`, `z_don_gia`:

| Alert tổng | Điều kiện |
|---|---|
| **CRITICAL** | ≥ 2 chỉ tiêu vượt WARN **hoặc** 1 chỉ tiêu ≥ CRIT |
| **WARN** | đúng 1 chỉ tiêu vượt WARN (và không CRITICAL) |
| **OK** | còn lại |

`drivers` liệt kê các chỉ tiêu góp phần vượt WARN — hỗ trợ review nghiệp vụ (lượng / trị giá / giá).

Ý nghĩa thiết kế: một spike đơn lẻ trên 1 metric dễ là nhiễu; lệch đồng thời lượng–trị giá (hoặc lệch cực mạnh một chiều) đáng tin hơn.

---

## 7. Đánh giá khi không có nhãn

Prototype dùng **synthetic injection**:

1. Giữ nguyên $\hat{y}_t$ và thang đo (median/MAD) từ backtest.
2. Nhân giá trị thực tế với hệ số $f$ trên thang gốc ↔ cộng $\log(f)$ trên thang log.
3. Tính lại $z$ giả lập; đo tỷ lệ tháng có $|z| \geq 3$.

Hệ số thử: `1.2, 1.5, 2.0, 3.0` (tăng) và `0.7, 0.5, 0.3` (giảm).

Đây là ước lượng **recall theo độ lớn shock**, không phải precision trên sự kiện thật. Bổ sung nên làm:

- Review top CRITICAL/WARN với sự kiện thị trường gạo / chính sách xuất khẩu.
- Temporal holdout (Dev/Holdout) nếu mở rộng production.
- Pseudo-label thủ công nếu cần P@n / AP.

Kết quả backtest được ghi ra `alert_backtest.csv`.

---

## 8. Tóm tắt kỹ thuật (cheat sheet)

| Thành phần | Kỹ thuật |
|---|---|
| Model dự báo | Holt-Winters / ETS additive + damped trend + seasonal 12 |
| Validation style | Walk-forward one-step (expanding/sliding window) |
| Transform | $\log(y)$ |
| Anomaly score | Robust z trên forecast residual |
| Scale robust | Rolling median + MAD (`SCALE_WIN=36`) |
| Ngưỡng | $\|z\|\geq 3$ WARN, $\|z\|\geq 4.5$ CRITICAL |
| Ensemble nhẹ | Rule gộp 3 metric (lượng, trị giá, đơn giá) |
| Đánh giá unlabeled | Synthetic multiplicative shock → recall |

---

## 9. File liên quan

| File | Vai trò |
|---|---|
| `rice_alert_prototype.py` | Script CLI chạy backtest |
| `rice_alert_prototype.ipynb` | Notebook + trực quan hóa |
| `alert_backtest.csv` | Output z-score & alert theo tháng |
| `docs/Plan.md` | Pipeline tổng thể dự án anomaly detection |

---

## 10. Hướng mở rộng gợi ý

1. Thêm baseline so sánh: robust MoM z / Isolation Forest trên feature lag–MoM.
2. Áp dụng cùng pipeline cho các mặt hàng KEEP trong registry eligibility.
3. Tuning `WARN`/`CRIT` theo phân vị holdout hoặc EVT.
4. Phân tách alert “giá” vs “lượng” rõ hơn trong báo cáo vận hành.
