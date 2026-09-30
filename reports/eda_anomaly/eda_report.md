# EDA chuỗi thời gian — Anomaly Detection

Nguồn: `data/csv/nhap_khau` + `data/csv/xuat_khau`. Phạm vi toàn cục: **2009-01 → 2026-08**.

## Tổng quan

- 17 mặt hàng–chiều → 34 chuỗi metric (`so_luong` / `tri_gia`).
- Coverage OK: **22**, có gap: **10**, insufficient: **2**.
- Candidate spikes (robust z ≥ 3.0 trên MoM): **282** điểm trên **282** sự kiện ghi nhận.

## Phương pháp

- Spike: `robust_z(|MoM|) >= 3.0 with rolling median/MAD window=12`.
- Mùa vụ: `1 - Var(y - month_mean)/Var(y)`.
- Candidate spikes are exploratory flags, not labeled anomalies.

## Coverage / missingness cần chú ý

| Series | Start–End | Valid | Absent | Null | Missing % | Status |
|---|---|---:|---:|---:|---:|---|
| `san_pham_tu_sat_thep__nhap_khau__so_luong` | 2009-01–2026-08 | 0 | 0 | 212 | 100.0% | INSUFFICIENT |
| `san_pham_tu_sat_thep__xuat_khau__so_luong` | 2009-01–2026-08 | 0 | 0 | 212 | 100.0% | INSUFFICIENT |
| `than_da__nhap_khau__tri_gia` | 2009-01–2026-08 | 48 | 0 | 164 | 77.4% | GAPS |
| `than_da__nhap_khau__so_luong` | 2009-01–2026-08 | 48 | 0 | 164 | 77.4% | GAPS |
| `than_da__xuat_khau__so_luong` | 2009-01–2026-08 | 97 | 0 | 115 | 54.2% | GAPS |
| `than_da__xuat_khau__tri_gia` | 2009-01–2026-08 | 97 | 0 | 115 | 54.2% | GAPS |
| `ure__nhap_khau__so_luong` | 2009-01–2026-08 | 106 | 0 | 106 | 50.0% | GAPS |
| `ure__nhap_khau__tri_gia` | 2009-01–2026-08 | 106 | 0 | 106 | 50.0% | GAPS |
| `dau_tho__nhap_khau__so_luong` | 2009-01–2026-08 | 154 | 0 | 58 | 27.4% | GAPS |
| `dau_tho__nhap_khau__tri_gia` | 2009-01–2026-08 | 154 | 0 | 58 | 27.4% | GAPS |
| `clanhke_va_xi_mang__xuat_khau__so_luong` | 2009-01–2026-08 | 164 | 0 | 48 | 22.6% | GAPS |
| `clanhke_va_xi_mang__xuat_khau__tri_gia` | 2009-01–2026-08 | 164 | 0 | 48 | 22.6% | GAPS |

## Chuỗi biến động mạnh nhất (CV, n≥36)

| Series | CV | Median (MoM) | Spikes z≥3 | Seasonal R² |
|---|---:|---:|---:|---:|
| `ure__nhap_khau__tri_gia` | 1.0656 | 0.6773 | 13.0 | 0.1016 |
| `ure__nhap_khau__so_luong` | 1.0227 | 0.6788 | 14.0 | 0.1142 |
| `sat_thep_cac_loai__xuat_khau__tri_gia` | 0.787 | 0.1104 | 8.0 | 0.0097 |
| `dau_tho__nhap_khau__tri_gia` | 0.7753 | 0.2358 | 12.0 | 0.0421 |
| `than_da__nhap_khau__so_luong` | 0.7521 | 0.3161 | 3.0 | 0.0227 |
| `than_da__xuat_khau__so_luong` | 0.7375 | 0.2861 | 8.0 | 0.0755 |
| `phan_bon_cac_loai__xuat_khau__tri_gia` | 0.735 | 0.1903 | 6.0 | 0.028 |
| `sat_thep_cac_loai__xuat_khau__so_luong` | 0.728 | 0.1123 | 10.0 | 0.0085 |
| `dau_tho__nhap_khau__so_luong` | 0.714 | 0.2129 | 15.0 | 0.0479 |
| `dau_tho__xuat_khau__tri_gia` | 0.6957 | 0.212 | 14.0 | 0.0105 |

## Đơn giá (trị giá / số lượng) — spike MoM

| Series | n | Median đơn giá | CV | Spikes |
|---|---:|---:|---:|---:|
| `xang_dau_cac_loai__xuat_khau__don_gia` | 212 | 676.95 | 0.2994 | 14 |
| `dau_tho__xuat_khau__don_gia` | 212 | 597.36 | 0.3132 | 12 |
| `xang_dau_cac_loai__nhap_khau__don_gia` | 200 | 691.78 | 0.3032 | 10 |
| `dau_tho__nhap_khau__don_gia` | 154 | 595.25 | 0.3026 | 9 |
| `phan_bon_cac_loai__xuat_khau__don_gia` | 188 | 368.06 | 0.2525 | 7 |
| `khi_dot_hoa_long__nhap_khau__don_gia` | 200 | 647.60 | 0.2853 | 7 |
| `ure__nhap_khau__don_gia` | 105 | 0.34 | 0.2823 | 7 |
| `sat_thep_cac_loai__xuat_khau__don_gia` | 212 | 727.55 | 0.1737 | 7 |
| `sat_thep_cac_loai__nhap_khau__don_gia` | 200 | 706.68 | 0.2193 | 6 |
| `phe_lieu_sat_thep__nhap_khau__don_gia` | 200 | 360.77 | 0.2166 | 6 |
| `phan_bon_cac_loai__nhap_khau__don_gia` | 200 | 324.23 | 0.1982 | 6 |
| `than_da__xuat_khau__don_gia` | 97 | 83.95 | 0.2592 | 6 |

## Gợi ý cho mô hình anomaly

1. **Tách train theo series** `loai × ten_mat_hang × metric`; không gộp chung một model nếu đơn vị khác nhau.
2. **Loại / cắt** chuỗi INSUFFICIENT hoặc missing cao (ure, phôi thép lượng, than đá kết thúc sớm) khỏi baseline unsupervised chung.
3. Feature hữu ích: MoM %, YoY %, residual sau seasonal month-mean, robust z, đơn giá, tỷ lệ lượng/trị giá lệch.
4. Spike z≥3 chỉ là **ứng viên** — cần xác nhận nghiệp vụ (kỳ nghỉ, thay đổi HS code, sửa số liệu).
5. Kiểm tra **đơn giá** cùng lúc với lượng/trị giá để tách anomaly giá vs anomaly khối lượng.

## Artifact

- `series_summary.csv`, `candidate_spikes.csv`, `unit_price_summary.csv`, `eda_payload.json`

```powershell
.\.venv\Scripts\python.exe -X utf8 scripts/eda_timeseries_anomaly.py
```