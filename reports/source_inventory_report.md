# Kiểm kê Silver `xuatnhapkhau` — bước 1

Phạm vi duy nhất: `xuatnhapkhau/`, theo yêu cầu người dùng. Báo cáo này thay thế bản kiểm kê bốn nguồn trước đó. Các kết luận so sánh giữa nguồn không còn thuộc bước 1.

Chỉ kiểm tra cấu trúc và chất lượng dữ liệu; không thay đổi Silver, nội suy, ghép nguồn hoặc huấn luyện mô hình. Chưa chốt Development/Holdout.

## Tổng quan

| File Parquet | Dòng | Partition mặt hàng–chiều | Tháng đầu | Tháng cuối |
|---:|---:|---:|---|---|
| 110 | 3063 | 17 | 2009-01 | 2026-08 |

Phạm vi tháng là toàn folder; từng partition có độ phủ khác nhau.

- Schema vật lý: `so_luong: double`, `tri_gia: double`, `thang_nam: date32[day]`.
- Chiều và mặt hàng nằm trong partition `loai=.../ten_mat_hang=...`.
- Không có cột đơn vị hoặc metadata xác nhận số tháng/lũy kế, thời điểm công bố.
- Lỗi đọc: 0; partition trùng tháng: 0; ngày không hợp lệ: 0.

## Ánh xạ phôi thép

Theo xác nhận của người dùng trong dự án, target `phoi_thep` nhập khẩu lấy từ partition:

`xuatnhapkhau/loai=nhap_khau/ten_mat_hang=sản phẩm từ sắt thép/`

Giữ nguyên nhãn và file Silver. `source_commodity=san_pham_tu_sat_thep`, còn tên target vẫn là `phoi_thep`. Đây là ánh xạ riêng của dự án do người dùng xác nhận, không phải quy tắc đồng nghĩa chung giữa hai nhóm hàng. Ánh xạ chỉ áp dụng cho chiều nhập khẩu.

## Độ phủ 16 tổ hợp mặt hàng–chiều / 32 target

| Target mặt hàng | Chiều | Tháng đầu–cuối | Tháng có | Tháng thiếu nội bộ | Null lượng / trị giá | Ghi chú |
|---|---|---|---:|---:|---|---|
| than_da | nhap_khau | 2013-01–2016-12 | 48 | 0 | 0 / 0 | Lịch sử kết thúc sớm |
| than_da | xuat_khau | 2009-01–2017-06 | 97 | 5 | 0 / 0 | Lịch sử kết thúc sớm |
| dau_tho | nhap_khau | 2010-01–2026-08 | 200 | 0 | 46 / 46 | 46 tháng khuyết mỗi chỉ tiêu, cuối cùng 2018-03 |
| dau_tho | xuat_khau | 2009-01–2026-08 | 212 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| xang_dau_cac_loai | nhap_khau | 2010-01–2026-08 | 200 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| xang_dau_cac_loai | xuat_khau | 2009-01–2026-08 | 212 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| phan_bon_cac_loai | nhap_khau | 2010-01–2026-08 | 200 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| phan_bon_cac_loai | xuat_khau | 2011-01–2026-08 | 188 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| sat_thep_cac_loai | nhap_khau | 2010-01–2026-08 | 200 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| sat_thep_cac_loai | xuat_khau | 2009-01–2026-08 | 212 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| khi_dot_hoa_long | nhap_khau | 2010-01–2026-08 | 200 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| ure | nhap_khau | 2009-06–2025-10 | 106 | 91 | 0 / 0 | Thiếu nhiều tháng |
| phoi_thep | nhap_khau | 2010-01–2026-08 | 200 | 0 | 200 / 0 | Ánh xạ đã xác nhận; khối lượng khuyết toàn bộ |
| phe_lieu_sat_thep | nhap_khau | 2010-01–2026-08 | 200 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| gao | xuat_khau | 2009-01–2026-08 | 212 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |
| clanhke_va_xi_mang | xuat_khau | 2013-01–2026-08 | 164 | 0 | 0 / 0 | Cần chốt đơn vị và định nghĩa |

Đã ánh xạ được 32/32 target tới partition. Có 1 target khuyết toàn bộ: `phoi_thep__nhap_khau__so_luong`.

Ánh xạ được nguồn không đồng nghĩa đủ điều kiện huấn luyện. Tháng thiếu nội bộ chỉ tính giữa tháng đầu và cuối của từng partition; chưa tính thiếu so với khoảng nghiệp vụ yêu cầu.

Partition `sản phẩm từ sắt thép` xuất khẩu vẫn được kiểm kê vì nằm trong folder, nhưng không thuộc 32 target yêu cầu và không được ánh xạ sang phôi thép xuất khẩu.

## Kết luận cho bước tiếp theo

1. Nguồn duy nhất của pipeline theo phạm vi hiện tại là `xuatnhapkhau`; registry sẽ tách tên target khỏi tên partition nguồn.
2. Phôi thép nhập: trị giá có dữ liệu; khối lượng khuyết 200/200 dòng nên gắn `BLOCKED_ALL_NULL`. Không thay null bằng 0 hoặc suy khối lượng từ trị giá.
3. Than nhập dừng ở 2016-12; than xuất dừng ở 2017-06 và thiếu 5 tháng nội bộ. Cần chốt phạm vi dự báo và cách xử lý độ phủ.
4. Urê có 106 tháng quan sát trong khoảng 2009-06–2025-10, thiếu 91 tháng nội bộ; chưa nội suy.
5. Dầu thô nhập có 46 tháng khuyết mỗi chỉ tiêu, tháng khuyết cuối 2018-03. Chưa tự cắt lịch sử.
6. Xác minh đơn vị từng partition, dữ liệu tháng/lũy kế, ý nghĩa null/0 và thời điểm công bố; chốt Development/Holdout trước khi phân tích hoặc lựa chọn mô hình.

## Đầu ra và tái lập

- `inventory.json`: manifest chỉ chứa file dưới `xuatnhapkhau`, SHA-256, schema và metadata.
- `partition_profiles.json`: toàn bộ partition của folder, tháng thiếu, tháng null và kiểm tra trùng.
- `target_candidates.json`: 32 target, tên mặt hàng nguồn, cơ sở ánh xạ, thống kê và trạng thái riêng theo chỉ tiêu.
- `source_comparisons.json`: danh sách rỗng vì phạm vi chỉ có một nguồn; thay thế kết quả so sánh cũ.
- `identical_partitions.json`: kiểm tra nội dung giống nhau giữa các partition trong folder duy nhất.

```powershell
.\.venv-audit\Scripts\python.exe -X utf8 scripts/audit_silver_sources.py
```

Dependencies kiểm kê: `scripts/requirements-audit.txt`. Môi trường `.venv-audit` phục vụ kiểm kê, chưa thiết lập môi trường huấn luyện ML.
