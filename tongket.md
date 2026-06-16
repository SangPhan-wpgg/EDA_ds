# Tổng kết — Phân tích chi tiết biểu đồ & tiền xử lý trong `yp2021_report.ipynb`

Ngày tổng kết: **16/06/2026**

Tài liệu này đọc kèm notebook báo cáo `notebooks/yp2021_report.ipynb` (sinh từ
`notebooks/_build_yp2021_report_nb.py`). Mục tiêu: với **mỗi biểu đồ**, nói rõ nó
trình bày gì, **kết luận rút ra dựa trên cơ sở số liệu/kiểm định nào**; và với
**phần tiền xử lý**, giải thích **tại sao mỗi bước lại được làm như thế**.

Mọi con số dưới đây lấy trực tiếp từ artefact pipeline (`reports/data_check/*.csv`)
nên trùng khớp với output notebook khi execute. Bài toán trọng tâm: dự đoán
`employed_w04` (có việc làm ở wave 4) trên **mẫu pilot 5.687 người** đã tốt nghiệp,
tỷ lệ có việc **p̂ = 0,7566** (4.302 có việc / 1.385 không việc / 0 khuyết).

> **Nguyên tắc diễn giải xuyên suốt:** mọi kết luận là **liên hệ/dự đoán, không
> nhân quả**; chỉ trong phạm vi **mẫu pilot wave 4 sau tốt nghiệp**; không suy rộng
> cho toàn panel YP2021 và không gộp với GOMS/KEEP II.

---

## Phần A — Bốn quyết định tiền xử lý nền tảng (tại sao làm như thế)

Trước khi phân tích từng biểu đồ, cần nắm bốn lựa chọn tiền xử lý chi phối **toàn
bộ** con số trong báo cáo. Đây là lý do "tại sao xử lý như thế" ở cấp toàn cục.

1. **Đọc lớp `processed`, không đọc thẳng `raw`.** Mọi biểu đồ EDA và modeling đọc
   từ `data/processed/youth_panel/pilot/yp2021_processed_pilot_w04_en.parquet`, chứ
   không đọc trực tiếp file `.dta` Stata. *Lý do:* các **mã bỏ qua (skip codes)**
   `9090908, 9090909, 9999997, 9999998, 9999999` chỉ được làm sạch ở lớp processed.
   Nếu đọc thô và coi các sentinel này là số thật, mọi trung bình/phương sai/tương
   quan sẽ sai lệch nghiêm trọng (ví dụ tuổi trung bình bị kéo lên hàng triệu). Lớp
   `raw_tables` chỉ dùng cho truy vết và cho phần *minh hoạ wrangling* (Mục 4).

2. **Lọc mẫu theo quy tắc `w04 == 1 and w04edu >= 3`.** YP2021 không phải khảo sát
   graduate-only mà tuyển thanh niên 19–28 tuổi nói chung. *Lý do lọc:* quần thể
   mục tiêu (P) của đồ án là người **sau tốt nghiệp CĐ/ĐH**, nên phải thu khung A
   (toàn panel) về đúng P bằng điều kiện đã tham gia wave 4 (`w04==1`) và học vấn ≥
   cao đẳng (`w04edu>=3`, với `3=전문대졸, 4=대졸, 5=석사이상`). Không lọc thì mẫu trộn
   lẫn người chưa tốt nghiệp → chệch bao phủ.

3. **Dùng bản nhãn tiếng Anh (`_en.parquet`).** Bản processed giữ cả mã gốc và nhãn
   giải mã; biểu đồ dùng bản `_en` để nhãn trục đọc được, tránh phụ thuộc render
   tiếng Hàn.

4. **Cấu hình phông `Malgun Gothic` + `axes.unicode_minus=False`.** *Lý do:* nhãn
   có thể lẫn ký tự Hàn; `Malgun Gothic` hiển thị được cả Hàn lẫn dấu tiếng Việt, và
   tắt `unicode_minus` để dấu trừ (ở skew, log-odds) không biến thành ô vuông tofu.

---

## Phần B — Mục 2: Phạm vi & P-A-S

### Biểu đồ 1 — Selection funnel từ Khung (A) đến Mẫu (S)

- **Trình bày:** biểu đồ cột 3 bậc: Khung wave 4 (12.213) → Tham gia wave 4 (9.665)
  → Sau tốt nghiệp `w04edu>=3` (5.687).
- **Cơ sở kết luận:** số đếm **quan sát trực tiếp** từ bảng
  `analysis_sample_funnel.parquet` (cột `respondent_count`), không phải số sao chép
  từ tài liệu ngoài. Hai bậc tụt lớn (−2.548 do attrition wave 4, −3.978 do lọc học
  vấn) cho thấy mẫu phân tích chỉ còn **46,6%** khung gốc → cảnh báo phải đọc kết quả
  trong phạm vi mẫu lọc, là bằng chứng định lượng cho **chệch bao phủ**.

### Biểu đồ 2 — Attrition panel wave 1 → wave 4

- **Trình bày:** đường tham gia qua 4 wave: 12.213 → 10.721 → 10.108 → 9.665.
- **Cơ sở kết luận:** lấy từ `panel_participation_summary.parquet`. Tỷ lệ giữ chân
  w1→w4 = 9.665/12.213 = **79,1%**, tức ~21% rơi rụng. *Cơ sở để lo ngại:* vì là dữ
  liệu **panel**, không phản hồi biểu hiện thành attrition tích luỹ — người rời panel
  có thể khác hệ thống so với người ở lại → đây là **chệch không phản hồi**, rủi ro
  phải ghi nhớ khi diễn giải mọi kết quả. Biểu đồ chỉ *mô tả* quy mô, không khẳng
  định hướng lệch.

> **Lưu ý Mục 2.1 (không có biểu đồ):** thiết kế tách **ba loại mẫu** — chính
> (5.687), nhạy cảm (~17% có khối ngành/GPA/loại trường), rà soát (job-history
> 4.932/1.717). Cơ sở tách: tiểu mẫu ~17% có tỷ lệ có việc ≈51,8% so với 75,6% toàn
> mẫu → trộn ngây thơ sẽ tiêm **chệch lựa chọn**. Đây là lý do nhiều biến sau này bị
> giữ ngoài model lõi.

---

## Phần C — Mục 3: Sai số lấy mẫu & bất định

### Biểu đồ 3 — Mô hình urn/Binomial chồng lên xấp xỉ Normal

- **Trình bày:** histogram tỷ lệ có việc từ 5.000 mô phỏng urn/Binomial(n, p̂), chồng
  đường mật độ Normal giải tích và vạch p̂.
- **Cơ sở kết luận:** ba cách ước lượng CI 95% cho tỷ lệ có việc **gần như trùng
  nhau** (`yp2021_sampling_variation_summary.csv`):
  - Giải tích (Normal approx): [0,7453, 0,7676], SE 0,00569
  - Urn/Binomial (5.000 sim): [0,7452, 0,7675]
  - Bootstrap (2.000): [0,7456, 0,7681]
  
  Kết luận "CI ≈ ±1,1 điểm phần trăm và xấp xỉ Normal đáng tin" có cơ sở **vì** cỡ
  mẫu lớn (n=5.687) và p̂ không sát 0/1 → định lý giới hạn trung tâm áp dụng, ba phương
  pháp hội tụ. Đây là kiểm chứng chéo, không chỉ dựa một công thức.

### Biểu đồ 4 — Bất định của chất lượng mô hình (2 panel)

- **Trình bày:** (trái) histogram bootstrap ROC-AUC trên test set; (phải) histogram
  ROC-AUC qua 200 lần split 80/20 ngẫu nhiên.
- **Cơ sở kết luận:**
  - Bootstrap ROC-AUC: điểm 0,6789, **CI 95% = [0,6407, 0,7146]** → khoảng **không
    chứa 0,5**. Đây là cơ sở khẳng định "tín hiệu phân loại là **thật**, không phải
    nhiễu lấy mẫu" — dù yếu (~0,68).
  - Split variation: mean 0,6813, **std chỉ 0,0170** → cơ sở khẳng định "mô hình
    **ổn định**, không phụ thuộc may rủi của một lần chia". Accuracy qua split: mean
    0,7587, std 0,0047.

---

## Phần D — Mục 4: Tiền xử lý & feature engineering từ raw (tại sao làm như thế)

Mục này **cố tình quay lại `raw_tables`** để phơi bày *quy trình* biến đổi (các mục
khác đọc processed đã sạch). Không có biểu đồ; thay vào đó là 6 nhóm kỹ thuật
wrangling — đây chính là phần "tại sao tiền xử lý như thế" mà đề bài yêu cầu.

| Kỹ thuật | Làm gì | **Tại sao làm như thế** |
|---|---|---|
| **Mask skip-code** | Biến `{9090908,…,9999999}` → `NaN` trên `w04edu, w04edu_f, w04edu_m, y04f308` | Sentinel là "câu hỏi bị bỏ qua/không trả lời", **không phải** giá trị số. Giữ nguyên sẽ bóp méo mean/var. Phải cô lập khuyết thật **trước** khi bàn điền khuyết. |
| **Điền khuyết 3 chiến lược** | (a) học vấn cha/mẹ → **cờ missing indicator**; (b) tuổi → **trung vị**; (c) tài sản hộ → **trung bình** làm tròn | Phân biệt theo *bản chất khuyết*: học vấn cha/mẹ khuyết theo **logic nhánh câu hỏi** (không ngẫu nhiên) → không bịa giá trị mà để model tự học qua cờ; tuổi điền **median** vì bền với lệch; tài sản là thang bậc → **mean** làm tròn. Điền sai bản chất = giả định MCAR vô căn cứ, tiêm nhiễu. |
| **Đổi granularity (`groupby`)** | `job_history` cấp **job-spell** (nhiều dòng/người) → cấp **cá nhân** (1 dòng/người), sinh `n_job_spells`, `years_active` | Phân tích & model ở cấp **người**, nên phải gom đợt-việc về người. Đây là bước đổi độ chi tiết kinh điển. |
| **Wide↔long (`melt`)** | Tham gia panel dạng rộng (mỗi wave 1 cột) → dạng dài (mỗi dòng = 1 respondent–wave) | Dạng **tidy/long** mới gom nhóm/vẽ attrition thuận tiện; đếm tham gia mỗi wave chỉ còn 1 dòng `groupby`. |
| **Rời rạc hóa (`pd.cut`)** | Tuổi liên tục → nhóm `22-24/25-27/28-30/31+` | Ranh giới chọn **theo phân phối quan sát ở EDA**, biến liên tục thành thứ bậc dễ đọc/đối chiếu subgroup. |
| **Regex** | Trích `^(\d{4})-(\d{1,2})$` từ trường ngày `"YYYY-MM"`; kiểm tháng hợp lệ `(0?[1-9]\|1[0-2])` | Minh hoạ trích xuất + **kiểm định tính hợp lệ cú pháp** (lớp ký tự, neo `^$`, định lượng `{4}`, nhóm, lựa chọn `\|`), phát hiện dòng nhiễm ký tự lạ. |
| **Mã hóa** | `get_dummies` one-hot **vs** `drop_first=True` (dummy) | One-hot giữ đủ mức; `drop_first` bỏ 1 mức/biến để **tránh đa cộng tuyến** với mô hình có hệ số chặn. Lựa chọn tùy mô hình hạ nguồn. |

---

## Phần E — Mục 5: EDA phân phối & coverage

### Biểu đồ 5 — Target + hai outcome job-history (3 panel)

- **Trình bày:** (1) cột phân phối `employed_w04` (Có việc 4.302 / Không 1.385);
  (2) histogram `job_changes`; (3) histogram `avg_job_duration_months`.
- **Cơ sở kết luận:** mẫu **nghiêng hẳn về phía có việc** (≈75,7%) — cơ sở để sau này
  *không* dùng accuracy đơn lẻ. Hai biến job-history **lệch phải mạnh** vì phần lớn
  người chỉ có một đợt việc (mode ở 0 lần đổi việc).

### Biểu đồ 6 — Lưới phân phối biến phân loại (11 ô, kèm % khuyết)

- **Trình bày:** small-multiples 11 biến phân loại, mỗi tiêu đề ghi tỷ lệ khuyết.
- **Cơ sở kết luận:** ưu tiên **độ phủ** hơn độ sâu — cho thấy nhóm ngành/GPA/loại
  trường/chuẩn bị việc có **% khuyết rất cao** (≈83%), trực quan hoá cơ sở cho việc
  loại chúng khỏi model lõi.

### Biểu đồ 7 — Lưới phân phối biến số (5 ô)

- **Trình bày:** histogram `age_w04, graduation_gpa_yp100, job_changes,
  avg_job_duration_months, total_jobs` kèm n hợp lệ.
- **Cơ sở kết luận:** n hợp lệ chênh lệch lớn (tuổi 5.687 vs GPA 946 vs duration
  1.717) là bằng chứng trực quan cho **chệch lựa chọn theo coverage**.

### Biểu đồ 8 — Tỷ lệ valid theo biến processed (đỏ < 50%)

- **Trình bày:** barh xếp theo độ phủ, vạch ngưỡng 50%, tô đỏ biến phủ < 50%.
- **Cơ sở kết luận:** lấy từ `yp2021_processed_coverage.parquet` (`valid_rate`). Các
  biến đỏ = ngành/loại trường/GPA/chuẩn bị việc, đều thuộc mô-đun "vừa tốt nghiệp đợt
  này" (~17% mẫu). **Hệ quả quyết định:** chúng *không* vào model lõi, chỉ dành cho
  model `--narrow-scope` có cờ chỉ báo khuyết. Đây là cầu nối coverage → quyết định
  thiết kế feature.

---

## Phần F — Mục 6: EDA chuyên sâu (Bernoulli/Binomial, phân phối, biến đổi)

### Bảng — Khung Bernoulli/Binomial + majority baseline (không phải biểu đồ)

- **Cơ sở:** `target_framing()` cho p̂=0,7565, n=5.687, Binomial kỳ vọng 4.302, std
  đếm 32,4. **Majority-baseline accuracy = 0,7565**. *Đây là cơ sở quan trọng nhất
  của toàn phần modeling:* mọi mô hình thật phải vượt mốc accuracy này, và vì baseline
  đã đạt accuracy cao nhờ mẫu lệch → phải đánh giá bằng **ROC-AUC/PR**, không chỉ
  accuracy.

### Bảng — Nhận diện phân phối lý thuyết

- **Cơ sở kết luận** (`yp2021_deep_eda_distribution_id.csv`): quy tắc Gauss-candidate
  |skew|<0,5 **và** |excess kurt|<1.
  - Tuổi: skew −0,084 nhưng kurt −1,042 → **không** Normal (quá phẳng, platykurtic).
  - GPA: skew −0,580; đổi việc 1,489; duration 1,890 → đều lệch rõ.
  - **Tất cả 5 biến đều non-normal.** Ghi chú đúng: Shapiro p cực nhỏ ở n lớn là bình
    thường (test quá nhạy) nên kết luận dựa **moment** (skew/kurt), không quyết bằng
    riêng p-value.

### Biểu đồ 9 — Ứng viên biến đổi (|skew| trước vs sau)

- **Trình bày:** barh so |skew| gốc và |skew| sau phép biến đổi đề xuất, vạch ngưỡng
  0,5.
- **Cơ sở kết luận** (`yp2021_deep_eda_transform_candidates.csv`): module thử log1p &
  sqrt rồi chọn phép cho |skew| nhỏ nhất:
  - `avg_job_duration_months`: 1,89 → **log1p** −0,34
  - `job_changes`: 1,49 → **sqrt** 0,68
  - `total_jobs`: 1,49 → **log1p** 1,00
  - tuổi (−0,08), GPA (−0,58) → **none** (biến đổi không cải thiện đủ −0,1)
  
  Đây là **đề xuất tiền-mô-hình-hóa**; lưu ý biến đếm việc vẫn bị loại khỏi model lõi
  vì leakage (xem Phần H), nên đề xuất chỉ áp cho phân tích phụ.

---

## Phần G — Mục 7: Liên hệ feature × target

### Biểu đồ 10 — Lưới tỷ lệ có việc theo nhóm phân loại (11 ô)

- **Trình bày:** barh tỷ lệ có việc theo từng mức của 11 biến (lọc nhóm n≥20), vạch
  nét đứt = TB chung 75,7%.
- **Cơ sở kết luận:** khoảng cách thanh so với đường chung = cường độ phân tách. Nhấn
  mạnh đúng: chênh lệch là **liên hệ trong mẫu**, không nhân quả.

### Bảng — Kiểm định thống kê (Welch t-test + chi-square)

- **Cơ sở kết luận** (`yp2021_eda_stat_tests.csv`), xếp theo p tăng dần:
  - **Tuổi** phân tách mạnh nhất: Welch t, p ≈ 3,9·10⁻⁷⁵ (TB 27,53 có việc vs 26,15
    không).
  - Job-history: job_changes/total_jobs p≈8·10⁻⁸, duration p≈4·10⁻⁷.
  - **Chứng chỉ** χ² p≈3,1·10⁻⁸; **ngành** p≈9,7·10⁻⁵; **học vấn cha** p≈1,7·10⁻³;
    học vấn p≈4,5·10⁻³; học vấn mẹ p≈6,9·10⁻³; giới tính p≈1,5·10⁻².
  - **Không đạt ý nghĩa 5%:** loại trường (0,097), tài sản cha mẹ (0,234), **GPA**
    (0,060), chuẩn bị việc GPA/spec (0,764) — phần do **cỡ mẫu hợp lệ nhỏ** (GPA chỉ
    946). Đây là cơ sở để *không* overclaim các biến này.

### Biểu đồ 11 — Heatmap tương quan Pearson (gồm target 0/1)

- **Trình bày:** ma trận Pearson giữa các biến số + target.
- **Cơ sở kết luận:** vừa cho thấy biến nào liên hệ outcome, vừa **cảnh báo cặp tự
  tương quan**. Bằng chứng cụ thể: `total_jobs ≈ job_changes + 1` → tương quan ≈1,0
  giữa hai biến (point-biserial của chúng với target trùng tới 4 chữ số: 0,0716) →
  cơ sở loại bớt khi model.

### Biểu đồ 12 — Xếp hạng liên hệ feature × outcome (thang chung)

- **Trình bày:** barh xếp hạng mọi feature theo |point-biserial r| (numeric) hoặc
  Cramér's V (categorical) — quy về **một thang cường độ chung**.
- **Cơ sở kết luận** (`yp2021_feature_target_association.csv`):
  1. **Tuổi 0,241** (mạnh nhất, n=5.687)
  2. ngành 0,152 *(nhưng n chỉ 948, hẹp ~17% → thận trọng)*
  3. duration 0,114 · vùng 0,079 · chứng chỉ 0,072 · total_jobs/job_changes 0,072 ·
     GPA 0,061 · loại trường 0,053 · học vấn cha 0,049 · mẹ 0,042 · học vấn 0,039 ·
     giới tính 0,029 · tài sản 0,019.
  
  Kết luận "tín hiệu khiêm tốn" có cơ sở: ngay biến mạnh nhất cũng chỉ r≈0,24 → giải
  thích vì sao ROC-AUC model chỉ ~0,68.

### Bảng — Subgroup ranking + Wilson CI + tầng độ chắc

- **Cơ sở kết luận** (`subgroup_ranking`): mỗi subgroup kèm **Wilson CI 95%** + tầng
  🔵/🟡/🟢. *Cơ sở phân tầng (kiểm soát overclaim):* 🔵 vững = χ² có ý nghĩa **và** CI
  không chứa tỷ lệ chung; 🟡 thận trọng = CI chồng chung **hoặc** thuộc subset chọn lọc
  ~17%; 🟢 = không khác chung. Ví dụ vùng Seoul 0,797 [0,772, 0,820] là 🔵; các mức
  ngành đều bị hạ 🟡 vì selection bias — chặn việc thổi phồng chênh lệch nhỏ thành
  "phát hiện".

### Biểu đồ 13 — Heatmap đa biến giữa predictor (Cramér's V)

- **Trình bày:** Cramér's V **giữa các predictor với nhau** (không phải với target).
- **Cơ sở kết luận:** cặp V cao = trùng thông tin → **đa cộng tuyến**, gợi ý lược
  bớt khi model. Bổ trợ cho cảnh báo `total_jobs ≈ job_changes + 1` ở Biểu đồ 11.

---

## Phần H — Mục 8: Feature catalog & leakage (không có biểu đồ)

- **Cơ sở:** catalog sinh từ `feature_catalog.build_catalog(df)`, nguồn là các tuple
  CORE/LEAKAGE/NARROW_SCOPE trong `dataset.py` nên **không thể lệch** khỏi tập feature
  model thực dùng.
- **Checklist chống rò rỉ — tại sao xử lý như thế:**
  - **Loại `total_jobs, job_changes, avg_job_duration_months`** khỏi cả core lẫn
    narrow: chúng đếm việc làm **sau** khi outcome được quan sát → dùng làm feature là
    leakage, làm điểm số "đẹp giả tạo".
  - **Không dùng cột nguồn target** (`employed_w04_source*`).
  - **Impute/scale/encode trong `Pipeline`**: chỉ fit trên fold train mỗi vòng CV →
    không rò thống kê test sang train.
  - **Chia trước, biến đổi sau** (`train_test_split` stratify chạy trước preprocessor).
  - **Cô lập narrow-scope:** khối ngành/GPA/loại trường (~17%) chỉ vào model
    `--narrow-scope` kèm cờ khuyết, không impute như MCAR.

---

## Phần I — Mục 9: Modeling & đánh giá nâng cao

### Bảng — Kiểm tra "modeling-ready" (9.1)

- **Cơ sở:** 4 kiểm tra hợp đồng dữ liệu chạy *trước* khi fit, có `assert`: (1) grain
  1 dòng/người (sampid không trùng); (2) target nhị phân 0/1 đã tách khỏi X; (3) không
  cột leakage trong X; (4) missingness < 100%/cột. Nếu một kiểm tra fail → dừng, đảm
  bảo mọi metric sau đó đáng tin.

### Bảng metric 4 mô hình (9.2)

- **Cơ sở** (`yp2021_model_metrics.csv`):

  | Mô hình | Acc | ROC-AUC | Avg.Prec | Brier | Recall |
  |---|---:|---:|---:|---:|---:|
  | baseline (most-frequent) | 0,757 | **0,500** | 0,757 | 0,243 | 1,000 |
  | logreg L2 | 0,760 | 0,679 | 0,855 | **0,170** | 0,983 |
  | logreg L1 | 0,761 | 0,679 | 0,855 | 0,170 | 0,984 |
  | random forest | 0,656 | **0,681** | 0,853 | 0,220 | 0,668 |

  *Đọc có cơ sở:* baseline accuracy 0,757 ≈ accuracy logistic, **nhưng** ROC-AUC=0,5 →
  baseline không phân biệt được ai thất nghiệp. Logistic vượt rõ về AUC/AP/Brier. RF
  có AUC nhỉnh nhất (0,681) nhưng accuracy/Brier kém hơn (đổi recall lấy precision).

### Mục 9.3 — Cầu nối toán học (markdown, không biểu đồ)

- **Cơ sở:** sigmoid σ(z), log-odds = z, **cross-entropy loss** (NLL Bernoulli),
  **gradient** ∂L/∂βⱼ = (1/n)Σ(p̂ᵢ−yᵢ)xᵢⱼ, baseline-theo-loss (MSE→mean, MAE→median,
  log-loss→prevalence). Giải thích *vì sao* baseline đúng là most-frequent và ROC-AUC
  của nó = 0,5.

### Biểu đồ 14 — Bốn ma trận nhầm lẫn

- **Trình bày:** confusion matrix test set cho 4 mô hình.
- **Cơ sở kết luận** (cột tn,fp,fn,tp của metrics):
  - baseline: (0, 277, 0, 861) → đoán "có việc" cho **tất cả** → bắt 0 người thất
    nghiệp. Đây là bằng chứng accuracy cao nhưng vô dụng với lớp thiểu số.
  - logreg L2: (19, 258, 15, 846) → bắt được 19/277 thất nghiệp.
  - random forest: (171, 106, 286, 575) → bắt **171/277** thất nghiệp (recall lớp 0
    tốt hơn hẳn) nhưng bỏ sót nhiều người có việc → accuracy tổng thấp hơn.
  
  *Cơ sở của thông điệp:* giá trị nằm ở khả năng nhận diện lớp thiểu số, điều accuracy
  đơn lẻ che giấu.

### Biểu đồ 15 — ROC + Precision-Recall (2 panel)

- **Cơ sở kết luận:** cả 3 mô hình thật đều **vượt rõ đường tham chiếu** (ROC chéo
  AUC=0,5; PR mức prevalence 0,757). AP ≈ 0,855 (logistic) ≫ prevalence 0,757 →
  xác nhận tín hiệu thật. PR phù hợp hơn ở đây vì lớp lệch (tập trung lớp dương).

### Biểu đồ 16 — Calibration + Threshold tuning (2 panel)

- **Cơ sở kết luận:**
  - Calibration: so xác suất dự đoán vs tần suất quan sát; **Brier** logistic 0,170 ≪
    baseline 0,243 → xác suất logistic hiệu chỉnh tốt hơn, đáng tin khi ra quyết định
    chứ không chỉ xếp hạng.
  - Threshold tuning: F1 theo ngưỡng (`yp2021_model_thresholds.csv`) cho thấy dời
    ngưỡng khỏi 0,5 đánh đổi precision–recall — cơ sở để chỉnh ngưỡng khi chi phí hai
    loại sai khác nhau.

### Biểu đồ 17 — Hiệu năng L2 theo nhóm (subgroup fairness)

- **Trình bày:** ROC-AUC mô hình L2 tách theo giới tính/vùng/học vấn (n≥30), vạch AUC
  tổng thể.
- **Cơ sở kết luận** (`results.subgroup_frame`, khớp `yp2021_model_subgroups.csv`):
  khoảng dao động (gap) ROC-AUC giữa các nhóm **nhỏ** → mô hình tương đối đồng đều, chưa
  thấy bất công bằng nghiêm trọng. Vẫn nhấn mạnh: là liên hệ trong mẫu pilot, không suy
  rộng. Ngưỡng n≥30 để mỗi ước lượng đủ ổn định.

### Biểu đồ 18 — Top 15 hệ số log-odds (L2)

- **Trình bày:** barh log-odds (xanh tăng / cam giảm khả năng có việc).
- **Cơ sở kết luận:** mỗi hệ số = log của tỉ số odds; **dấu và độ lớn nhất quán với
  liên hệ ở EDA** (tuổi/chứng chỉ/vùng/nền tảng học vấn cha mẹ nổi bật) → khép kín
  mạch khám phá → mô hình, tăng độ tin cậy diễn giải.

### Bảng — Đánh đổi bias–variance (9.6)

- **Cơ sở kết luận** (`metrics_frame`):
  - baseline: CV train = CV test = 0,5 → **bias cao** (underfit).
  - logistic L2: CV train AUC 0,694 vs CV test 0,674 → **gap hẹp ≈0,02** → điều chuẩn
    kiểm soát phương sai tốt, không overfit.
  - random forest: CV train 0,754 vs CV test 0,671 → **gap rộng ≈0,08** (variance cao
    hơn) nhưng test AUC tương đương logistic.
  
  *Kết luận có cơ sở:* RF không thắng dù phức tạp hơn → **trần thông tin của tập
  feature** (sau khi loại leakage) mới là yếu tố giới hạn, không phải dạng mô hình.

---

## Phần J — Tổng hợp cơ sở kết luận chung

| Kết luận chính của báo cáo | Cơ sở số liệu/kiểm định |
|---|---|
| ~¾ người tốt nghiệp có việc ở wave 4 | p̂=0,7566 (4.302/5.687); CI 95% [0,745, 0,768] trùng nhau 3 phương pháp |
| Tín hiệu phân loại là thật nhưng yếu | Bootstrap ROC-AUC CI [0,641, 0,715] loại trừ 0,5; AUC ≈0,68 |
| Mô hình ổn định, không phụ thuộc 1 lần chia | Split variation std 0,017 (AUC), 0,005 (acc) |
| Yếu tố liên hệ mạnh nhất: tuổi, chứng chỉ, vùng, nền tảng cha mẹ | point-biserial/Cramér's V + Welch t/χ² (p<0,05); log-odds L2 nhất quán |
| Không overclaim ngành/GPA/loại trường | n hợp lệ nhỏ (≤948), p>0,05 hoặc tier 🟡 selection bias |
| Loại 3 biến job-history khỏi model lõi | Đo *sau* outcome → leakage (catalog + checklist) |
| Logistic ổn hơn RF dù đơn giản hơn | Gap train–CV hẹp (0,02 vs 0,08), test AUC tương đương |

**Giới hạn cần đọc kèm mọi biểu đồ:** lọc học vấn → chệch bao phủ; attrition 21% →
chệch không phản hồi; tiểu mẫu ~17% → chệch lựa chọn (gắn cờ 🟡); job-history hồi tưởng
+ GPA phân loại chuẩn hoá riêng → chệch đo lường. Mọi kết luận: **liên hệ/dự đoán,
không nhân quả**, chỉ trong **pilot wave 4 sau tốt nghiệp**, không gộp 3 dataset.
