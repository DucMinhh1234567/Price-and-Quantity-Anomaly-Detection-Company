# Spec: feature causal thu gọn (8–12 cột)

Nguồn: `scripts/build_causal_features.py` → `data/features/causal/*_causal.csv`.

## Nguyên tắc

- **Causal:** mọi baseline rolling dùng `shift(1)` trước cửa sổ — tháng \(t\) không nhìn \(t\) trở đi khi ước lượng median/MAD.
- **Thu gọn:** đúng **10** feature (trong khoảng 8–12 theo `Claude.pdf`).
- **Phạm vi:** 4 chuỗi XK đủ 212 tháng (gạo, sắt thép các loại, xăng dầu các loại, dầu thô). Không gồm sản phẩm từ sắt thép.

## Cột đã chọn

| # | Cột | Nhóm | Lý do |
|---|---|---|---|
| 1 | `log_so_luong_mom` | Thay đổi | log-MoM lượng |
| 2 | `log_so_luong_yoy` | Thay đổi | log-YoY lượng |
| 3 | `log_tri_gia_mom` | Thay đổi | log-MoM trị giá |
| 4 | `log_tri_gia_yoy` | Thay đổi | log-YoY trị giá |
| 5 | `log_don_gia_mom` | Thay đổi | log-MoM đơn giá |
| 6 | `log_don_gia_yoy` | Thay đổi | log-YoY đơn giá |
| 7 | `so_luong_robust_z12` | So với quá khứ | robust z 12 tháng (causal) |
| 8 | `tri_gia_robust_z12` | So với quá khứ | robust z 12 tháng (causal) |
| 9 | `don_gia_robust_z12` | So với quá khứ | robust z 12 tháng (causal) |
| 10 | `z_don_gia_minus_z_so_luong` | Giá vs lượng | lệch cấu trúc giá–khối lượng |

Cột định danh giữ kèm (không đưa vào model): `loai`, `ten_mat_hang`, `so_luong`, `tri_gia`, `don_gia`, `source_file`.

## Không đưa vào bản compact (để sau / dư thừa)

- Lag 1/2/3/12, rolling mean/std nhiều cửa sổ (~66 cột exploratory trong `data/features/`)
- Tổng 3 tháng YoY, YTD, sin/cos tháng, cờ Tết, residual ETS làm feature

## Chạy lại

```powershell
python -X utf8 scripts/build_causal_features.py
```
