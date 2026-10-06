# Hệ thống cảnh báo bất thường giá và lượng — Các bước từ đầu đến cuối

Tóm tắt theo `Claude.pdf`. Phạm vi dữ liệu: chuỗi xuất nhập khẩu hàng tháng (2009-01 → 2026-08).

**Phase hiện tại (thử nghiệm):** chỉ benchmark phương pháp + metric trên **gạo** (và tối đa vài chuỗi XK đủ 212 tháng). Chưa dựng hệ thống cảnh báo vận hành.

**Kết luận thử nghiệm (sau Bước 1–5):** winner tạm thời là **ETS + robust z**. Chronos-Bolt và Isolation Forest chưa đạt win rule trên gạo.

### Notebook quan sát (không nhân đôi logic)

| Phase | Notebook | Đọc / gọi |
|---|---|---|
| 1 ETS | [`scripts/ets_tier1_explore.ipynb`](../scripts/ets_tier1_explore.ipynb) | `reports/alert_backtest/` + `rice_alert_prototype.py` |
| 2 Feature | [`scripts/causal_features_explore.ipynb`](../scripts/causal_features_explore.ipynb) | `data/features/causal/` |
| 4 Chronos | [`scripts/compare_chronos_ets.ipynb`](../scripts/compare_chronos_ets.ipynb) | `reports/eval/chronos_*` |
| 5 IF/ECOD | [`scripts/benchmark_if_ecod.ipynb`](../scripts/benchmark_if_ecod.ipynb) | `reports/eval/if_ecod_*` |

Mở notebook từ VS Code/Cursor với kernel Python của project; working directory nên là `scripts/` hoặc root (cell tự resolve `ROOT`).


> Kết luận ngắn: dữ liệu gốc không cần thêm/bớt dòng, nhưng cần chuẩn hoá. Không có mô hình “tốt nhất” duy nhất — baseline thống kê vẫn cạnh tranh; foundation model (Chronos-2) đáng thử thêm.

**Chú thích trạng thái**

| Ký hiệu | Ý nghĩa |
|---|---|
| ✅ Đã làm | Có artifact / code / báo cáo trong repo |
| 🟡 Một phần | Có một phần, còn thiếu hoặc chưa đủ cho thí nghiệm |
| ❌ Chưa làm | Còn trong scope thử nghiệm, chưa làm |
| ⏸️ Để sau | Ngoài scope phase hiện tại — làm sau khi chốt model/metric |
| 🚫 Không thử | Có chủ đích không làm (không phù hợp chuỗi ~212 điểm) |

---

## Bảng tổng quan tiến độ

| Giai đoạn | Tiến độ | Scope phase này |
|---|---|---|
| 0. Chốt phạm vi & thứ tự chuỗi | ✅ | Trong scope (đã xong) |
| 1. Chuẩn bị dữ liệu (tối thiểu) | 🟡 | Trong scope; vintage + biến ngoài ⏸️ |
| 2. Feature engineering (thu gọn, causal) | ✅ | `data/features/causal/` + `docs/feature_causal_spec.md` |
| 3. Benchmark model theo tầng | ✅ | ETS thắng tạm vs Chronos-Bolt & IF trên gạo; conformal ⏸️ |
| 4. Đánh giá metric | ✅ | Protocol + baseline ETS + báo cáo so sánh |
| 5. Việc tiếp theo | 🟡 | 5A gần xong; còn sự kiện thật gạo (tuỳ chọn) |

---

## Giai đoạn 0 — Chốt phạm vi và thứ tự chuỗi

**Trạng thái giai đoạn: ✅**

### 0.1. Ưu tiên chuỗi theo độ đầy đủ

| Mức | Chuỗi | Việc cần làm | Trạng thái |
|---|---|---|---|
| Đủ 212/212 tháng | Xuất khẩu: **gạo**, sắt thép các loại, xăng dầu các loại, dầu thô | Làm trước. Gạo đầu tiên | ✅ Đã xác định; gạo đã có prototype |
| Gần đủ (~200 tháng) | Nhập khẩu: xăng dầu, khí đốt hóa lỏng, phân bón, phế liệu sắt thép, sắt thép các loại | Mở rộng sau khi protocol ổn | ⏸️ Để sau |
| Ngắn hơn | Xuất khẩu phân bón (188), clanhke và xi măng (164) | Làm sau nhóm đủ dữ liệu | ⏸️ Để sau |
| Coverage kém | Nhập khẩu dầu thô, than đá, ure | Chạy riêng đoạn liên tục | ⏸️ Để sau |
| Không dùng | Sản phẩm từ sắt thép (nhập và xuất) | Không có dữ liệu hợp lệ | ✅ Đã loại (INSUFFICIENT / BLOCKED) |

**Thứ tự đề xuất (dài hạn):** gạo → sắt thép XK → xăng dầu XK → dầu thô XK → ⏸️ 5 chuỗi NK 200 tháng.

Bằng chứng trong repo: `reports/source_inventory_report.md`, `reports/data_diagnostics/`, `reports/eda_anomaly/`, `docs/Plan.md`.

### 0.2. Chốt đơn vị phát hiện

| Việc | Trạng thái | Bằng chứng |
|---|---|---|
| Theo từng chuỗi `loai × mặt_hàng × metric` (+ đơn giá) | ✅ | `Plan.md`, EDA, prototype |
| Loại bất thường: spike MoM, level shift, giá lệch khi lượng bình thường | ✅ | Đã ghi trong plan / docs |
| Registry eligibility KEEP / CUT / EXCLUDE | ✅ | `diagnostics_report.md` (23 KEEP, …) |

---

## Giai đoạn 1 — Chuẩn bị dữ liệu (không thêm dòng mới)

**Trạng thái giai đoạn: 🟡** (phần tối thiểu cho thí nghiệm)

### 1.1. Chuẩn hoá bắt buộc

| # | Việc | Trạng thái | Ghi chú |
|---|---|---|---|
| 1 | Lịch tháng liên tục; giữ NaN, không nội suy | 🟡 | Prototype gạo: `asfreq("MS")`. Diagnostics đo missing; chưa pipeline clean thống nhất mọi chuỗi |
| 2 | Tạo `don_gia = tri_gia / so_luong` | ✅ | Có trong features CSV, EDA unit price, `rice_alert_prototype.py` |
| 3 | Lấy **log** cho lượng, trị giá, đơn giá | 🟡 | Chỉ trong ETS prototype (`np.log`); file `data/features/*` vẫn thang gốc |
| 4 | Xác nhận đơn vị + tháng vs lũy kế | 🟡 | Semantics: **MONTHLY_LIKELY**. Đơn vị gạo (tấn/USD) mới suy luận, chưa xác nhận nguồn |
| 5 | Lưu vintage nếu nguồn chỉnh số sau công bố | ⏸️ Để sau | Không cần cho benchmark model hiện tại |

### 1.2. Biến ngoài (exogenous) — ⏸️ Để sau

> Toàn bộ mục này **ngoài scope thử nghiệm**. Thêm sau khi đã chốt model thắng baseline (giảm false alarm vận hành).

| Biến | Trạng thái |
|---|---|
| Giá dầu Brent | ⏸️ Để sau |
| Tỷ giá USD/VND | ⏸️ Để sau |
| Giá gạo quốc tế / giá thép | ⏸️ Để sau |
| Lịch: tháng, số ngày làm việc, cờ Tết | ⏸️ Để sau |

---

## Giai đoạn 2 — Feature engineering

**Trạng thái giai đoạn: 🟡**

### 2.1. Chọn hướng theo loại mô hình

| Loại mô hình | Cần feature? | Trạng thái |
|---|---|---|
| Sai số dự báo (ETS…) | Gần như không | ✅ Đã dùng cho gạo |
| Vector đặc trưng (IF, ECOD, LOF…) | 8–12 feature | ✅ Matrix causal 10 cột sẵn; IF/ECOD còn ở Bước 5 |

### 2.2. Nhóm feature gợi ý

| Nhóm | Feature | Trạng thái |
|---|---|---|
| Thay đổi | MoM, YoY, lag, diff | ✅ Exploratory đầy đủ + **log-MoM/YoY** trong `data/features/causal/` |
| So với quá khứ | robust z / median-MAD 6–12 | ✅ Causal `*_robust_z12` trong compact set |
| Log-MoM / log-YoY | theo khuyến nghị PDF | ✅ Trong `data/features/causal/` |
| Giá và lượng | `z(giá)−z(lượng)`; đơn giá lệch median | ✅ `z_don_gia_minus_z_so_luong` |
| Dồn nhiều tháng | tổng 3 tháng YoY; YTD vs năm trước | ⏸️ Để sau | Sau khi so xong model điểm tháng |
| Từ mô hình khác | residual ETS/SARIMA làm feature | ⏸️ Để sau | Chỉ cần khi làm ensemble / tầng 3 phức tạp hơn |
| Lịch | sin/cos tháng, cờ Tết, ngày làm việc | ⏸️ Để sau | Đi cùng biến ngoài |

**File đã lưu:** `gạo`, `xăng dầu các loại`, `sắt thép các loại`, `dầu thô`, `sản phẩm từ sắt thép` — trong đó sản phẩm sắt thép **không nên dùng** để train.

### 2.3. Quy tắc chống rò rỉ

| Việc | Trạng thái | Ghi chú |
|---|---|---|
| Option `causal_baseline=True` (`shift(1)`) trong notebook | ✅ | Đã code sẵn |
| Feature CSV đã xuất dùng cho so sánh model | ✅ | Causal compact: `data/features/causal/` (10 cột). Spec: `docs/feature_causal_spec.md`. Exploratory `causal_baseline=False` vẫn ở `data/features/` |
| Prototype ETS không nhìn tương lai | ✅ | `rice_alert_prototype.py` dùng cửa sổ `[:t]` + MAD `shift(1)` |

---

## Giai đoạn 3 — Benchmark mô hình theo tầng

**Trạng thái giai đoạn: 🟡**  
*(Đổi tên từ “xây hệ thống cảnh báo” → phase này chỉ **so phương pháp**, chưa deploy.)*

### 3.1. Ứng viên thí nghiệm

| Tầng | Phương pháp | Trạng thái | Chi tiết |
|---|---|---|---|
| **Tầng 1** | ETS + robust z | ✅ | Đã chạy 4 chuỗi XK: gạo, sắt thép, xăng dầu, dầu thô. Báo cáo: `reports/alert_backtest/ets_tier1_comparison.md` |
| **Tầng 2** | Chronos-2 zero-shot + phân vị | 🟡 | Chronos-2-small không tải được; đã chạy fallback `chronos-bolt-small` (q10–q90) trên gạo. **Không thắng ETS** (recall cao nhưng alerts/năm 3.85 ≫ 0.15). Báo cáo: `reports/eval/chronos_vs_ets_gao.md` |
| **Tầng 3** | Isolation Forest / ECOD trên feature nhỏ | ✅ | IF + ECOD walk-forward trên 4 chuỗi causal. **IF không thắng ETS** trên gạo. Báo cáo: `reports/eval/if_ecod_vs_ets.md` |
| **Kết hợp production** | Rank/percentile + **conformal** | ⏸️ Để sau | Chỉnh ngưỡng vận hành sau khi chọn winner |

> Chỉ giữ tầng nào thắng baseline trong backtest dự án.

### 3.2. Không khuyến nghị

- LSTM-AE / Anomaly Transformer / TimesNet — 🚫 **Không thử** với chuỗi ~212 điểm (theo PDF / TSB-AD).

---

## Giai đoạn 4 — Đánh giá (hai lớp)

**Trạng thái giai đoạn: 🟡** — đây là trọng tâm phase thử nghiệm.

### 4.1. Chất lượng dự báo *(cho residual / Chronos)*

| Metric | Trạng thái |
|---|---|
| MASE | 🟡 | Hàm sẵn trong `scripts/eval_anomaly_protocol.py`; bảng ETS chờ yhat export |
| Độ phủ khoảng tin cậy | 🟡 | Hàm sẵn; áp dụng khi model có khoảng (Chronos) |
| Pinball loss | 🟡 | Hàm sẵn; áp dụng khi có quantile forecast |

### 4.2. Chất lượng phát hiện *(quyết định model nào tốt hơn)*

| Metric / việc | Trạng thái | Ghi chú |
|---|---|---|
| Recall trên bất thường giả lập | 🟡 | Có trên 4 chuỗi XK trong `ets_tier1_comparison.md` |
| Số cảnh báo mỗi năm | 🟡 | Có trên 4 chuỗi XK (gạo ~0.15, xăng dầu ~1.08 WARN+CRIT/năm) |
| PR-AUC, precision@k | 🟡 | `precision_at_k` có trong protocol; PR-AUC khi có nhãn/ranking đủ |
| Danh sách sự kiện thật (gạo) — sanity check | ❌ | Còn trong scope (không bắt buộc đủ nhãn) |
| VUS-PR (khi có nhãn đầy đủ) | ⏸️ Để sau | Chưa có nhãn; thước đo TSB-AD khi có label |

### 4.3. Backtest ETS tier-1 — ✅ 4 chuỗi XK

- Gạo: 1 CRITICAL (2025-10), 1 WARN; recall lượng x2 ≈ 0.43
- Sắt thép: nhạy hơn gạo (WARN+CRIT/năm ≈ 0.46); CRITICAL 2026-07
- Xăng dầu: nhiều CRITICAL nhất (9), gồm cụm COVID 2020 và 2026 — WARN+CRIT/năm ≈ 1.08
- Dầu thô: nhiều CRITICAL theo `don_gia` (2020, 2024) — WARN+CRIT/năm ≈ 1.0
- Chi tiết: `reports/alert_backtest/`; chạy lại: `python -X utf8 rice_alert_prototype.py --batch`

---

## Giai đoạn 5 — Việc tiếp theo

### 5A. Trong scope thử nghiệm (làm tiếp)

| # | Việc | Trạng thái | Việc cụ thể còn lại |
|---|---|---|---|
| 1 | So sánh ETS vs Chronos-2 trên gạo XK | 🟡 | Đã so Chronos-Bolt vs ETS; Chronos-2 chưa tải được. Kết luận tạm: **giữ ETS**. Xem `reports/eval/chronos_vs_ets_gao.md` |
| 2 | Chạy tầng 1 trên sắt thép, xăng dầu, dầu thô XK | ✅ | Đã chạy `--batch`; xem `reports/alert_backtest/ets_tier1_comparison.md` |
| 3 | Xuất feature causal + thu gọn 8–12 cột; thử IF/ECOD | ✅ | Feature causal + IF/ECOD benchmark xong; winner tạm = ETS |
| 4 | Chốt bộ metric + quy tắc “thắng baseline” | ✅ | `scripts/eval_anomaly_protocol.py` + `reports/eval/baseline_ets.md` |
| 5 | Danh sách vài sự kiện thật của gạo | ❌ | Đối chiếu CRITICAL/WARN (sanity check) |

### 5B. ⏸️ Để sau (vận hành / mở rộng — ngoài phase này)

| # | Việc | Trạng thái |
|---|---|---|
| 1 | Thêm cảnh báo tổng 3 tháng cho lượng | ⏸️ Để sau |
| 2 | Biến ngoài (Brent, tỷ giá, giá quốc tế, Tết) | ⏸️ Để sau |
| 3 | Conformal calibration ngưỡng | ⏸️ Để sau |
| 4 | Ensemble gộp điểm production | ⏸️ Để sau |
| 5 | Chốt kênh nhận cảnh báo (email / Slack / dashboard / HTML) | ⏸️ Để sau |
| 6 | Mở rộng 5 chuỗi NK ~200 tháng và chuỗi ngắn hơn | ⏸️ Để sau |

---

## Việc nên làm ngay (theo độ ưu tiên) — chỉ phase thử nghiệm

1. ~~Chạy tầng 1 trên 3 chuỗi XK~~ ✅
2. ~~Feature causal thu gọn~~ ✅
3. ~~Chronos vs ETS~~ 🟡 (Bolt xong; Chronos-2 chưa tải được)
4. ~~Chốt metric + IF/ECOD~~ ✅ — **winner tạm: ETS**
5. **(Tuỳ chọn)** vài sự kiện thật gạo để sanity-check CRITICAL 2025-10

> ⏸️ Sau khi chốt model: tổng 3 tháng → biến ngoài → conformal → kênh alert → chuỗi NK.

---

## Tóm tắt lộ trình

```
Phase thử nghiệm (hiện tại)
  ├─ Chuẩn hoá tối thiểu + feature causal thu gọn     ✅
  ├─ ETS trên 4 chuỗi XK                             ✅
  ├─ Chronos-Bolt vs ETS trên gạo                    🟡 (không thắng; Chronos-2 chưa có)
  ├─ IF/ECOD vs ETS                                  ✅ (không thắng trên gạo)
  └─ Winner tạm: ETS + robust z                      ✅

⏸️ Để sau (không làm trong phase này)
  ├─ Vintage, biến ngoài, feature lịch / YTD / 3 tháng
  ├─ Conformal + ensemble production
  ├─ Kênh cảnh báo vận hành
  └─ Mở rộng chuỗi NK / chuỗi ngắn / VUS-PR có nhãn

🚫 Không thử
  └─ LSTM-AE, Anomaly Transformer, TimesNet
```

---

## Nguồn tham khảo (từ PDF)

- Liu & Paparrizos (2024), TSB-AD — [OpenReview](https://openreview.net/forum?id=R6kJtWsTGy)
- Liu, Lee & Paparrizos, TSB-AutoAD — [PVLDB](https://www.vldb.org/pvldb/vol18/p4364-liu.pdf)
- Ansari et al., Chronos-2 — [arXiv](https://arxiv.org/pdf/2510.15821)
- Reverso: Efficient Time Series Foundation Models — [arXiv](https://arxiv.org/pdf/2602.17634)
- Post-Training in Time Series Foundation Models (§7.4 Anomaly Detection) — [arXiv](https://arxiv.org/pdf/2607.20002)
