# Step 1B — Chẩn đoán missingness

Nguồn duy nhất: `xuatnhapkhau/`. Ánh xạ đọc từ `artifacts/source_inventory/target_candidates.json`; đây là kết quả khám phá, chưa phải registry chính thức.

## Định nghĩa

- Khoảng kiểm tra: lịch tháng liên tục từ tháng nhỏ nhất đến lớn nhất có bản ghi trong từng partition nguồn, bao gồm cả bản ghi null. Không đo thiếu trước/sau hai mốc này.
- `missingness.csv` chỉ chứa tháng có vấn đề, một dòng cho mỗi target × tháng. Tháng bình thường không xuất hiện.
- `MISSING_MONTH`: không có bản ghi tháng. `NULL_VALUE`: có bản ghi nhưng chỉ tiêu khuyết. Hai loại không đếm trùng.
- `expected_months`: số tháng của lịch đầy đủ. `observed_months`: số tháng có bản ghi, kể cả bản ghi null.
- `missing_month_count`: tháng không có bản ghi. `null_count`: tháng có bản ghi nhưng giá trị null.
- `first_missing`, `last_missing`: tháng đầu/cuối có một trong hai vấn đề; để trống nếu không thiếu.
- `longest_missing_run`: chuỗi tháng liên tiếp dài nhất có một trong hai vấn đề, kể cả chuyển từ tháng vắng sang tháng null.
- `missing_ratio = (missing_month_count + null_count) / expected_months`, số thực từ 0 đến 1.
- Giá trị 0 không coi là thiếu. Không nội suy, điền 0, cắt lịch sử, cộng gộp hoặc huấn luyện mô hình. Gặp tháng trùng hoặc ngày không hợp lệ thì dừng.

## Kết quả

Đã kiểm tra 32 target; 484 dòng vấn đề. Số đếm theo target nên cùng một tháng có thể xuất hiện riêng cho lượng và trị giá.

| Target | Tháng vắng | Null | Đầu thiếu | Cuối thiếu | Chuỗi thiếu dài nhất | Tỷ lệ thiếu |
|---|---:|---:|---|---|---:|---:|
| dau_tho__nhap_khau__so_luong | 0 | 46 | 2010-01 | 2018-03 | 7 | 23.00% |
| dau_tho__nhap_khau__tri_gia | 0 | 46 | 2010-01 | 2018-03 | 7 | 23.00% |
| phoi_thep__nhap_khau__so_luong | 0 | 200 | 2010-01 | 2026-08 | 200 | 100.00% |
| than_da__xuat_khau__so_luong | 5 | 0 | 2017-01 | 2017-05 | 5 | 4.90% |
| than_da__xuat_khau__tri_gia | 5 | 0 | 2017-01 | 2017-05 | 5 | 4.90% |
| ure__nhap_khau__so_luong | 91 | 0 | 2009-07 | 2025-09 | 16 | 46.19% |
| ure__nhap_khau__tri_gia | 91 | 0 | 2009-07 | 2025-09 | 16 | 46.19% |

Các target không thiếu vẫn có dòng trong `missingness_summary.csv` với số đếm bằng 0.

Phôi thép nhập khẩu được ánh xạ tới `sản phẩm từ sắt thép` theo xác nhận của người dùng. Tên target vẫn giữ `phoi_thep`; không áp ánh xạ này cho chiều xuất.

Tỷ lệ thiếu bằng 0 không chứng minh chuỗi đủ mới để dự báo: cần xem riêng `start_month` và `end_month`, đặc biệt với than đá.

## Giới hạn và bước tiếp theo

Đây là kiểm tra chất lượng toàn lịch sử, chưa lựa chọn mô hình hoặc quyết định xử lý dựa trên Holdout. Cần chốt temporal firewall trước khi phân tích mô hình.

Đã kiểm tra tháng/lũy kế cho nhóm eligible bên dưới; chưa kiểm tra đơn giá hoặc chốt đơn vị. Urê vẫn chờ xác nhận ý nghĩa tháng thiếu, chưa có chính sách điền 0.

## Chạy lại

```powershell
.\.venv-audit\Scripts\python.exe -X utf8 scripts/diagnose_silver_series.py
```

Dùng dependencies có sẵn trong `scripts/requirements-audit.txt`. CSV mã hóa UTF-8 BOM; tháng dạng YYYY-MM; tỷ lệ lưu dạng số, không kèm ký hiệu %. `diagnostics_manifest.json` lưu mã hash input và ánh xạ dùng trong lần chạy.

## Registry và eligibility

`configs/series_registry.csv` ghi các quyết định đã được người dùng chốt; script chỉ đọc, không tự cập nhật status từ kết quả thống kê. `target_candidates.json` giữ nguyên là artifact khám phá.

| Status | Số target |
|---|---:|
| KEEP | 23 |
| EXCLUDED_COVERAGE | 4 |
| CUT_HISTORY | 2 |
| NEED_BUSINESS_CONFIRMATION | 2 |
| BLOCKED_ALL_NULL | 1 |

23 KEEP + 2 CUT_HISTORY = 25 target được kiểm tra semantics. Eligibility này chưa phải xác nhận cuối cùng cho training. Hai target dầu thô nhập chỉ dùng cửa sổ từ 2018-04 cho kiểm tra cumulative; missingness phía trên vẫn đo toàn Silver.

## Monthly / cumulative — phương pháp

- Không biến đổi giá trị. Chỉ xét năm đủ 12 tháng có dữ liệu hữu hạn; bỏ năm đầu/cuối chưa đủ tháng. Danh sách năm và khoảng phân tích được lưu trong CSV.
- `years_monotonic`: số năm có 11 chênh lệch tháng liên tiếp đều >= 0 (không giảm). `monotonic_ratio = years_monotonic / years_checked`.
- `year_reset_ratio`: tỷ lệ biên năm có tháng 1 nhỏ hơn tháng 12 ngay trước đó; mẫu số chỉ gồm các cặp năm đầy đủ liên tiếp, lưu ở `year_boundaries_checked`. Không có biên hợp lệ thì để trống.
- Cần tối thiểu 3 năm đầy đủ và ít nhất 80% năm không hằng số. Cửa sổ có tháng vắng, null, vô hạn hoặc giá trị âm được đánh dấu INCONCLUSIVE.
- MONTHLY_LIKELY: monotonic_ratio <= 0.2. CUMULATIVE_LIKELY: monotonic_ratio >= 0.8 và year_reset_ratio >= 0.8 trên ít nhất 2 biên năm. Các trường hợp khác: INCONCLUSIVE.
- Đây là ngưỡng heuristic đặt trước khi chạy, không phải kiểm định chứng minh semantics. Năm giảm trong chuỗi lũy kế có điều chỉnh vẫn có thể xảy ra; reset đầu năm một mình cũng không chứng minh lũy kế.

| Target | Năm đủ | Năm không giảm | Tỷ lệ không giảm | Tỷ lệ reset | Assessment |
|---|---:|---:|---:|---:|---|
| clanhke_va_xi_mang__xuat_khau__so_luong | 13 | 0 | 0.000 | 0.583 | MONTHLY_LIKELY |
| clanhke_va_xi_mang__xuat_khau__tri_gia | 13 | 0 | 0.000 | 0.500 | MONTHLY_LIKELY |
| dau_tho__nhap_khau__so_luong | 7 | 0 | 0.000 | 0.500 | MONTHLY_LIKELY |
| dau_tho__nhap_khau__tri_gia | 7 | 0 | 0.000 | 0.333 | MONTHLY_LIKELY |
| dau_tho__xuat_khau__so_luong | 17 | 0 | 0.000 | 0.625 | MONTHLY_LIKELY |
| dau_tho__xuat_khau__tri_gia | 17 | 0 | 0.000 | 0.500 | MONTHLY_LIKELY |
| gao__xuat_khau__so_luong | 17 | 0 | 0.000 | 0.625 | MONTHLY_LIKELY |
| gao__xuat_khau__tri_gia | 17 | 0 | 0.000 | 0.688 | MONTHLY_LIKELY |
| khi_dot_hoa_long__nhap_khau__so_luong | 16 | 0 | 0.000 | 0.467 | MONTHLY_LIKELY |
| khi_dot_hoa_long__nhap_khau__tri_gia | 16 | 0 | 0.000 | 0.333 | MONTHLY_LIKELY |
| phan_bon_cac_loai__nhap_khau__so_luong | 16 | 0 | 0.000 | 0.867 | MONTHLY_LIKELY |
| phan_bon_cac_loai__nhap_khau__tri_gia | 16 | 0 | 0.000 | 0.800 | MONTHLY_LIKELY |
| phan_bon_cac_loai__xuat_khau__so_luong | 15 | 0 | 0.000 | 0.643 | MONTHLY_LIKELY |
| phan_bon_cac_loai__xuat_khau__tri_gia | 15 | 0 | 0.000 | 0.643 | MONTHLY_LIKELY |
| phe_lieu_sat_thep__nhap_khau__so_luong | 16 | 0 | 0.000 | 0.867 | MONTHLY_LIKELY |
| phe_lieu_sat_thep__nhap_khau__tri_gia | 16 | 0 | 0.000 | 0.867 | MONTHLY_LIKELY |
| phoi_thep__nhap_khau__tri_gia | 16 | 0 | 0.000 | 0.800 | MONTHLY_LIKELY |
| sat_thep_cac_loai__nhap_khau__so_luong | 16 | 0 | 0.000 | 0.600 | MONTHLY_LIKELY |
| sat_thep_cac_loai__nhap_khau__tri_gia | 16 | 0 | 0.000 | 0.600 | MONTHLY_LIKELY |
| sat_thep_cac_loai__xuat_khau__so_luong | 17 | 0 | 0.000 | 0.625 | MONTHLY_LIKELY |
| sat_thep_cac_loai__xuat_khau__tri_gia | 17 | 0 | 0.000 | 0.625 | MONTHLY_LIKELY |
| xang_dau_cac_loai__nhap_khau__so_luong | 16 | 0 | 0.000 | 0.533 | MONTHLY_LIKELY |
| xang_dau_cac_loai__nhap_khau__tri_gia | 16 | 0 | 0.000 | 0.533 | MONTHLY_LIKELY |
| xang_dau_cac_loai__xuat_khau__so_luong | 17 | 0 | 0.000 | 0.500 | MONTHLY_LIKELY |
| xang_dau_cac_loai__xuat_khau__tri_gia | 17 | 0 | 0.000 | 0.562 | MONTHLY_LIKELY |

Tất cả target được kiểm tra là MONTHLY_LIKELY. Bước tiếp theo là chốt Development/Holdout; chưa chạy training.
