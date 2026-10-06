Đây là **pipeline end-to-end** cho bài toán của bạn: chuỗi tháng xuất/nhập khẩu, **không nhãn**, 2 metric chính (`so_luong`, `tri_gia`), nhiều mặt hàng. Bạn đã xong phần lớn bước 0–1 (inventory + EDA); phần dưới gắn với dữ liệu thực tế trong repo.

---

## 0. Định nghĩa bài toán (trước khi code model)

Chốt rõ:

| Mục | Gợi ý cho dataset của bạn |
|---|---|
| Đơn vị phát hiện | Theo **từng chuỗi**: `loai × mặt_hàng × metric` (hoặc thêm `don_gia`) |
| Loại anomaly | Point spike (MoM), level shift, giá bất thường khi lượng bình thường |
| Không gộp 1 model cho mọi mặt hàng | Đơn vị khác nhau, CV khác nhau (ure CV ~1.0 vs sắt thép ổn hơn) |
| Không nhãn | Unsupervised + review nghiệp vụ; pseudo-label sau nếu cần |

**Eligibility** (đã có trong diagnostics): KEEP / CUT_HISTORY / EXCLUDE. Ví dụ: bỏ `sản phẩm từ sắt thép` lượng (100% null), than đá lịch sử ngắn, ure thiếu nhiều tháng.

---

## 1. EDA (đã gần xong — củng cố trước train)

Làm **theo từng series**, không gộp:

1. **Coverage**: start/end, % null, longest missing run  
2. **Semantics**: tháng vs lũy kế (báo cáo của bạn: `MONTHLY_LIKELY` — tốt)  
3. **Phân phối**: histogram, CV, skew; so sánh mức tuyệt đối vs MoM  
4. **Mùa vụ**: month-mean / STL nhẹ (seasonal R² trong EDA khá thấp → mùa vụ yếu)  
5. **Đơn giá** `tri_gia / so_luong`: tách anomaly **giá** vs **khối lượng**  
6. **Ứng viên spike** (robust z trên MoM): chỉ là candidate, chưa phải ground truth  

Output EDA nên có: bảng series đủ/không đủ train, plot đường + candidate spikes, bảng đơn giá.

---

## 2. Cleaning / chuẩn bị chuỗi

Thứ tự thực tế:

1. Parse `thang_nam` → index tháng đều  
2. Drop cột hằng (`loai`, `ten_mat_hang` khi train từng file)  
3. **Chính sách missing** (quan trọng với data của bạn):  
   - Đầu chuỗi null (vd. sắt thép 2009): **cắt** tới tháng có dữ liệu  
   - Null giữa chuỗi (dầu thô nhập tới 2018-03): **cắt lịch sử** hoặc mask, **không** điền 0  
   - Không nội suy nếu mục tiêu là phát hiện bất thường (nội suy dễ “làm đẹp” anomaly)  
4. Loại giá trị vô lý: `so_luong ≤ 0` khi tính đơn giá → `don_gia = NaN`  
5. Chốt cửa sổ train: ví dụ Development đến 2023-12, Holdout 2024-01 → 2026-08 (**temporal firewall** — đã ghi trong diagnostics)

---

## 3. Feature engineering

Nhóm feature hữu ích cho loại data này:

**Mức gốc**
- `so_luong`, `tri_gia`, `don_gia = tri_gia / so_luong`

**Biến động thời gian**
- MoM %, YoY %  
- Diff bậc 1: `Δx_t = x_t - x_{t-1}`  
- Lag 1, 2, 3, 12  

**Thống kê trượt (robust)**
- Rolling median / MAD cửa sổ 6–12  
- Robust z: `(x - median) / (1.4826 * MAD)`  

**Mùa vụ / residual**
- `x - mean(tháng trong năm)` hoặc residual STL  

**Đa biến nhẹ (cùng tháng)**
- Lệch đồng thời lượng vs trị giá (lượng tăng mạnh nhưng đơn giá không đổi / ngược lại)

Scale bằng **median–MAD** hoặc RobustScaler; tránh StandardScaler nếu chuỗi đã có outlier.

---

## 4. Chọn & train model (unsupervised)

Train **riêng từng series** (hoặc nhóm cùng đơn vị). Baseline nên có 2–3 lớp:

| Nhóm | Model | Phát hiện gì |
|---|---|---|
| Statistical / SPC | Robust control chart, CUSUM | Spike điểm, shift mức |
| Forecast residual | AR / Holt-Winters / Prophet nhẹ | Điểm lệch dự báo |
| Proximity | Isolation Forest, LOF, kNN trên feature cửa sổ | Điểm “lạ” trong không gian feature |
| Multivariate | PCA reconstruction error trên `[so_luong, tri_gia, don_gia]` | Lệch cấu trúc lượng–giá |

Khuyến nghị thực dụng: **robust z / control chart + Isolation Forest trên feature MoM/lag** làm ensemble (score trung bình hoặc max rank).

Mỗi model → **anomaly score** theo tháng → ngưỡng top 1–3% hoặc EVT / fixed robust z ≥ 3.

---

## 5. Kiểm thử & đánh giá (không có nhãn)

Vì unlabeled, đánh giá theo tầng:

**A. Temporal holdout**  
Fit chỉ trên Development; score trên Holdout — xem có “bùng” giả do drift không.

**B. Consistency / stability**  
- Cùng điểm có bị nhiều model flag không?  
- Score có ổn khi đổi cửa sổ MAD / contamination không?

**C. Domain review**  
Top-k điểm → đối chiếu sự kiện (COVID 2020, giá dầu, chính sách thương mại) + candidate spikes từ EDA.

**D. Pseudo-label (nếu cần metric số)**  
Gán nhãn thủ công ~20–50 điểm rõ → tính **P@n, R@n, AP** trên ranking score.

**E. Phân loại false positive thường gặp**  
Tháng Tết / nghỉ, đổi HS code, sửa số liệu, đơn vị đổi giữa kỳ.

Không dùng accuracy; ưu tiên ranking + precision trên top alert.

---

## 6. Thứ tự làm việc đề xuất (checklist)

```
1. Registry eligibility (KEEP / CUT / EXCLUDE)     ← gần xong
2. EDA + unit price + candidate spikes            ← gần xong
3. Clean theo series + chốt Dev/Holdout
4. Feature matrix (MoM, lag, robust z, don_gia)
5. Baseline SPC (robust z / CUSUM)
6. Isolation Forest / LOF trên cùng feature
7. Ensemble score + threshold
8. Review top alerts + (optional) pseudo-label metrics
9. Báo cáo: series nào train được, FP pattern, model thắng
```

---

## Điểm cần nhớ với đúng dataset của bạn

- **17 mặt hàng–chiều**, ~34 chuỗi metric; coverage không đều → **không train chung một model**.  
- `sản phẩm từ sắt thép` lượng = insufficient; than đá / ure = coverage kém.  
- Dữ liệu **tháng (không lũy kế)** → MoM/YoY hợp lệ.  
- Spike z≥3 trong EDA chỉ là **ứng viên**, không phải y_true.  
- Luôn xem **đơn giá** song song lượng/trị giá.

Nếu bạn muốn bước tiếp theo cụ thể hơn, có thể đi sâu một trong ba: (1) schema feature matrix chuẩn, (2) cách chốt Dev/Holdout + threshold, hoặc (3) pseudo-code Isolation Forest + robust z cho một mặt hàng (vd. dầu thô / sắt thép).