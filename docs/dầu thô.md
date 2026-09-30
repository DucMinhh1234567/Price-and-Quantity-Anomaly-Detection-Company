Dựa trên cấu trúc tệp **`dầu_thô.csv`** (gồm dữ liệu xuất khẩu dầu thô theo chuỗi thời gian hàng tháng từ năm 2009 đến 2026 với hai chỉ số định lượng chính là sản lượng `so_luong` và trị giá `tri_gia`) cùng các phương pháp được hệ thống trong chuỗi tài liệu **Anomaly Detection**, quy trình từng bước để xử lý bài toán phát hiện bất thường cho tệp này bao gồm 5 giai đoạn:

---

### **1. Tiền xử lý dữ liệu (Data Preprocessing)**
* **Định dạng chỉ mục thời gian (Time Series Indexing)**: Chuyển đổi cột `thang_nam` sang kiểu `datetime` và thiết lập làm chỉ mục (index) với các khoảng thời gian đều đặn theo tháng.
* **Loại bỏ đặc trưng không phân biệt (Feature Filtering)**: Cột `loai` (`xuat_khau`) và `ten_mat_hang` (`dầu thô`) chỉ chứa các giá trị hằng số cố định, cần loại bỏ để giảm nhiễu cho các thuật toán.
* **Chuẩn hóa dữ liệu (Scaling & Standardization)**: Vì `so_luong` và `tri_gia` có quy mô dữ liệu rất lớn và đơn vị khác biệt, cần thực hiện chuẩn hóa:
  * **Z-score**: Đưa dữ liệu về dạng \\(z = \frac{x - \mu}{\sigma}\\).
  * **Modified Z-score (Khuyên dùng)**: Do giá trị trung bình (\\(\mu\\)) và độ lệch chuẩn (\\(\sigma\\)) dễ bị chênh lệch bởi các điểm ngoại lệ sẵn có trong dữ liệu, việc dùng Trung vị (Median) và **MAD (Median Absolute Deviation)** sẽ bền vững (robust) hơn đối với nhiễu.

---

### **2. Kỹ thuật tạo đặc trưng (Feature Engineering)**
* **Tạo đặc trưng Đơn giá (Unit Price)**: Bổ sung cột mới \\(\text{don\_gia} = \frac{\text{tri\_gia}}{\text{so\_luong}}\\) để theo dõi biến động giá xuất khẩu đơn thuần, giúp phát hiện sự bất thường về giá độc lập với sản lượng.
* **Đặc trưng độ trễ (Lag Features)**: Tạo các biến trễ \\(X_{t-1}, X_{t-2}, \dots\\) thể hiện giá trị ở các tháng trước đó để phục vụ mô hình tự hồi quy.
* **Thống kê trượt (Rolling Statistics)**: Tính trung bình trượt (rolling mean) và độ lệch chuẩn trượt (rolling std) trong cửa sổ 3 hoặc 6 tháng để nắm bắt xu hướng (trend) và tính mùa vụ (seasonality).
* **Lấy sai phân (Differencing)**: Nếu chuỗi thời gian có xu hướng biến động dài hạn, lấy sai phân \\(\Delta X_t = X_t - X_{t-1}\\) để đưa dữ liệu về dạng dừng trước khi đưa vào mô hình.

---

### **3. Lựa chọn Mô hình & Thuật toán (Model Selection & Scoring)**
Tệp dữ liệu này mang bản chất là **chuỗi thời gian đa biến không gán nhãn (Unsupervised Time Series)**, do đó có thể kết hợp các nhóm mô hình sau:

1. **Chuỗi thời gian & Biểu đồ kiểm soát (Time Series / SPC)**:
   * **Control Chart**: Thiết lập ngưỡng quyết định \\(\mu \pm z \cdot \sigma\\) (hoặc Median \\(\pm z \cdot \text{MAD}\\)) cho `so_luong`, `tri_gia` và `don_gia`. Các điểm nằm ngoài ngưỡng \\(z\\) được xác định là bất thường điểm (point anomaly).
   * **Mô hình Tự hồi quy AR(p) / ARMA**: Dự báo giá trị tháng \\(X_t\\) dựa vào các tháng trước \\(X_{t-1}, \dots, X_{t-p}\\). Mức độ bất thường được đánh giá qua **residuals (sai số dự báo)** \\(e_t = |X_t - \hat{X}_t|\\).
   * **CUSUM (Cumulative Sum)**: Tính tổng tích lũy sai lệch để phát hiện thời điểm dịch chuyển xu hướng hoặc thay đổi mức độ hoạt động dài hạn (change points).
2. **Phương pháp Tuyến tính (Linear Methods / PCA)**:
   * **PCA (Principal Component Analysis)**: Biến đổi `so_luong` và `tri_gia` sang các thành phần chính. Điểm không nằm trong không gian con chứa phần lớn phương sai (subspace) hoặc có sai số tái tạo cao sẽ nhận điểm bất thường cao.
3. **Phương pháp Cận kề & Mật độ (Proximity-Based Methods)**:
   * **k-NN (k-Nearest Neighbors)**: Đo khoảng cách tới \\(k\\) điểm lân cận.
   * **LOF (Local Outlier Factor)**: So sánh mật độ cục bộ của một tháng so với các tháng xung quanh (\\(LOF > 1\\) chỉ ra khu vực có mật độ thưa thớt bất thường).

---

### **4. Chấm điểm & Thiết lập Ngưỡng (Scoring & Thresholding)**
* Mỗi thuật toán sẽ trả về một **Anomaly Score** đại diện cho mức độ bất thường của từng tháng.
* **Xác định Ngưỡng**: Chọn ngưỡng theo tỷ lệ phần trăm phân phối (ví dụ: top 1%–3% điểm cao nhất) hoặc áp dụng **Extreme Value Analysis (EVT)** để chọn ngưỡng tối ưu từ phân phối cực trị.

---

### **5. Kiểm thử & Đánh giá Chất lượng Mô hình (Model Evaluation)**
Vì `dầu_thô.csv` là dữ liệu thực tế **chưa có nhãn sẵn (Unlabeled)**, quá trình đánh giá thực hiện như sau:

1. **Thẩm định theo Bối cảnh Thực tế (Domain Knowledge Review)**:
   * Đối chiếu các thời điểm nhận Anomaly Score cao nhất với các sự kiện kinh tế - xã hội lịch sử (như khủng hoảng tài chính 2008–2009, đại dịch Covid-19 năm 2020, hoặc các biến động chính trị làm thay đổi giá dầu thế giới).
2. **Đánh giá theo Thứ hạng (Rank-based Evaluation)** *(Khi thực hiện gán nhãn thử nghiệm trên một tập mẫu)*:
   * **Precision at n (\\(P@n\\))**: Tỷ lệ phát hiện chính xác bất thường trong \\(n\\) tháng có điểm bất thường cao nhất.
   * **Recall at n (\\(R@n\\))**: Tỷ lệ tìm thấy bất thường so với tổng số điểm ngoại lệ trong top \\(n\\).
   * **Average Precision (AP)** & **ROC AUC**: Đánh giá khả năng xếp hạng của mô hình trên dữ liệu mất cân bằng lớp mà không bị phụ thuộc vào một ngưỡng tham số cố định \\(n\\).

---

💡 Bạn có muốn tôi chạy thử một kịch bản phân tích thực tế bằng Python trực tiếp trên tệp `dầu_thô.csv` (ví dụ: áp dụng Control Chart hoặc CUSUM để tìm các tháng bất thường) không?