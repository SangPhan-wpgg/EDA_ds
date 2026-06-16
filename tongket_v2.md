# Tổng kết — Phân tích chi tiết biểu đồ, tiền xử lý & logic chọn biến trong `yp2021_report.ipynb`

Ngày tổng kết: **16/06/2026**

Tài liệu đọc kèm `notebooks/yp2021_report.ipynb` (sinh từ `_build_yp2021_report_nb.py`).
Với **mỗi biểu đồ** trình bày 4 mục: **Loại** · **Ý nghĩa / phân tích điều gì** ·
**Cơ sở kết luận** (số liệu/kiểm định cụ thể). Hai phần riêng giải thích **tại sao tiền
xử lý như thế** và **tại sao chọn trường này, bỏ trường kia vào mô hình**.

Mọi con số lấy từ artefact pipeline (`reports/data_check/*.csv`,
`yp2021_processed_coverage.parquet`, `yp2021_feature_catalog.csv`). Bài toán: dự đoán
`employed_w04` trên **pilot 5.687 người** đã tốt nghiệp; tỷ lệ có việc **p̂ = 0,7566**
(4.302 có / 1.385 không / 0 khuyết).

> **Nguyên tắc diễn giải:** mọi kết luận là **liên hệ/dự đoán, không nhân quả**; chỉ
> trong **pilot wave 4 sau tốt nghiệp**; không suy rộng toàn panel, không gộp GOMS/KEEP.

---

## Phần A — Bốn quyết định tiền xử lý nền tảng (tại sao làm như thế)

1. **Đọc lớp `processed`, không đọc `raw`.** Mọi EDA/model đọc
   `yp2021_processed_pilot_w04_en.parquet`. *Lý do:* skip-code `9090908, 9090909,
   9999997, 9999998, 9999999` chỉ được mask ở processed; đọc thô coi sentinel là số thật
   → mọi mean/var/tương quan sai lệch. Raw chỉ dùng để truy vết và minh hoạ wrangling.
2. **Lọc `w04==1 & w04edu>=3`.** YP2021 tuyển thanh niên 19–28 nói chung; phải thu khung
   về đúng quần thể **sau tốt nghiệp CĐ/ĐH** (`3=전문대졸, 4=대졸, 5=석사이상`). Không lọc →
   chệch bao phủ.
3. **Bản nhãn tiếng Anh (`_en`).** Giữ cả mã gốc + nhãn giải mã; dùng `_en` để nhãn trục
   đọc được.
4. **Phông `Malgun Gothic` + `axes.unicode_minus=False`.** Hiển thị được cả Hàn lẫn dấu
   Việt; tránh dấu trừ (skew, log-odds) thành ô vuông tofu.

---

## Phần B — Mục 2: Phạm vi & P-A-S

### Biểu đồ 1 — Selection funnel A → S
- **Loại:** cột dọc 3 bậc (bar chart).
- **Ý nghĩa / phân tích gì:** mức thu hẹp từ Khung chọn mẫu (A) về Mẫu phân tích (S),
  định lượng từng bước rơi.
- **Cơ sở kết luận:** đếm trực tiếp từ `analysis_sample_funnel.parquet`: 12.213 → 9.665
  (−2.548 attrition) → 5.687 (−3.978 lọc học vấn). S chỉ còn **46,6%** khung gốc → bằng
  chứng định lượng cho **chệch bao phủ**; buộc đọc kết quả trong phạm vi mẫu lọc.

### Biểu đồ 2 — Attrition panel w1→w4
- **Loại:** đường (line chart), trục x = wave.
- **Ý nghĩa / phân tích gì:** rơi rụng tích luỹ của panel qua 4 đợt.
- **Cơ sở kết luận:** `panel_participation_summary.parquet`: 12.213 → 10.721 → 10.108 →
  9.665. Giữ chân w1→w4 = **79,1%** (~21% rơi). Vì là panel, không phản hồi = attrition
  → **chệch không phản hồi**: người rời có thể khác hệ thống người ở lại. Biểu đồ chỉ
  *mô tả quy mô*, không khẳng định hướng lệch.

> **Mục 2.1 (bảng, không biểu đồ):** tách **3 loại mẫu** — chính (5.687) / nhạy cảm
> (~17%) / rà soát (job-history). *Cơ sở tách:* tiểu mẫu ~17% có tỷ lệ có việc ≈51,8% vs
> 75,6% toàn mẫu → trộn ngây thơ tiêm **chệch lựa chọn**.

---

## Phần C — Mục 3: Sai số lấy mẫu & bất định

### Biểu đồ 3 — Urn/Binomial vs Normal
- **Loại:** histogram (mật độ) chồng đường mật độ Normal + vạch dọc p̂.
- **Ý nghĩa / phân tích gì:** phân phối lấy mẫu của tỷ lệ có việc nếu khảo sát lặp lại;
  kiểm tra xấp xỉ Normal có đáng tin không.
- **Cơ sở kết luận:** 3 CI 95% **trùng nhau** (`sampling_variation_summary.csv`): giải
  tích [0,7453; 0,7676]; urn/Binomial [0,7452; 0,7675]; bootstrap [0,7456; 0,7681].
  Trùng vì n lớn (5.687) + p̂ xa 0/1 → định lý giới hạn trung tâm áp dụng. CI ≈ **±1,1
  điểm %** → ước lượng tỷ lệ chắc chắn.

### Biểu đồ 4 — Bất định chất lượng mô hình (2 panel)
- **Loại:** 2 histogram (bootstrap AUC | AUC qua nhiều split).
- **Ý nghĩa / phân tích gì:** độ "rung lắc" của ROC-AUC do lấy mẫu và do chia train/test.
- **Cơ sở kết luận:** Bootstrap ROC-AUC 0,6789, **CI [0,6407; 0,7146] không chứa 0,5** →
  tín hiệu phân loại **thật** (dù yếu). Split variation: mean 0,6813, **std 0,0170** (acc
  std 0,0047) → mô hình **ổn định**, không phụ thuộc một lần chia may rủi.

---

## Phần D — Mục 4: Tiền xử lý từ raw (tại sao làm như thế)

Mục này **cố tình quay lại `raw_tables`** để phơi bày *quy trình*. Không có biểu đồ; là
6 nhóm kỹ thuật wrangling — phần "tại sao xử lý như thế" cốt lõi.

| Kỹ thuật | Làm gì | **Tại sao** |
|---|---|---|
| **Mask skip-code** | `{9090908…9999999}` → `NaN` | Sentinel = "bỏ qua/không trả lời", không phải số thật → phải cô lập khuyết *trước* khi điền. |
| **Điền khuyết 3 chiến lược** | học vấn cha/mẹ → **cờ missing**; tuổi → **median**; tài sản → **mean** làm tròn | Phân biệt *bản chất khuyết*: học vấn cha/mẹ khuyết theo **nhánh câu hỏi** (không ngẫu nhiên) → để model học qua cờ, không bịa; tuổi median bền với lệch; tài sản thang bậc → mean. Điền sai = giả định MCAR vô căn cứ. |
| **`groupby` đổi granularity** | job-spell (nhiều dòng/người) → cấp **cá nhân** | Phân tích & model ở cấp người. |
| **`melt` wide→long** | tham gia panel rộng → dài (1 dòng = respondent–wave) | Dạng tidy mới gom nhóm/vẽ attrition gọn. |
| **`pd.cut` rời rạc hóa** | tuổi → `22-24/25-27/28-30/31+` | Ranh giới chọn **theo phân phối EDA**; thành thứ bậc dễ đối chiếu. |
| **Regex** | `^(\d{4})-(\d{1,2})$` trên ngày `"YYYY-MM"` | Trích xuất + **kiểm hợp lệ cú pháp** (lớp ký tự/neo/định lượng/nhóm/lựa chọn). |
| **Mã hóa** | one-hot **vs** `drop_first=True` | one-hot giữ đủ mức; drop_first bỏ 1 mức/biến **tránh đa cộng tuyến** với model có hệ số chặn. |

---

## Phần E — Mục 5: EDA phân phối & coverage

### Biểu đồ 5 — Target + 2 outcome job-history (3 panel)
- **Loại:** 1 cột (target) + 2 histogram.
- **Ý nghĩa / phân tích gì:** hình dáng biến mục tiêu và hai outcome phụ.
- **Cơ sở kết luận:** target **lệch về có việc** (4.302 vs 1.385 ≈ 75,7%) → cơ sở *không*
  đánh giá bằng accuracy đơn lẻ. job_changes/duration **lệch phải mạnh** (mode ở 0/1 đợt
  việc).

### Biểu đồ 6 — Lưới phân phối biến phân loại (11 ô, kèm % khuyết)
- **Loại:** small-multiples barh ngang.
- **Ý nghĩa / phân tích gì:** ưu tiên *độ phủ* — quét toàn bộ nhóm biến trong 1 hình.
- **Cơ sở kết luận:** tiêu đề mỗi ô ghi % khuyết → thấy ngành/GPA/loại trường/chuẩn bị
  việc **khuyết ≈83%** (chỉ ~17% phủ), trực quan hoá lý do loại khỏi model lõi.

### Biểu đồ 7 — Lưới phân phối biến số (5 ô)
- **Loại:** small-multiples histogram.
- **Ý nghĩa / phân tích gì:** hình dáng + n hợp lệ của biến số.
- **Cơ sở kết luận:** n chênh lớn (tuổi 5.687 vs GPA 946 vs duration 1.717) = bằng chứng
  **chệch lựa chọn theo coverage**.

### Biểu đồ 8 — Tỷ lệ valid theo biến processed (đỏ < 50%)
- **Loại:** barh ngang, tô màu phân kỳ theo ngưỡng 50%.
- **Ý nghĩa / phân tích gì:** **bản đồ độ phủ** toàn bộ biến — đầu vào trực tiếp cho quyết
  định chọn feature.
- **Cơ sở kết luận** (`yp2021_processed_coverage.parquet`): biến đỏ = `job_prep_gpa_spec`
  13,6% · `graduation_gpa`/`grad_univ_type` 16,6% · `major` 16,7% · `avg_job_duration`
  30,2%. **Hệ quả:** không vào model lõi (xem Phần H–I). *Đây là biểu đồ then chốt cho
  yêu cầu "chọn/bỏ trường".*

---

## Phần F — Mục 6: EDA chuyên sâu

### Bảng — Bernoulli/Binomial + majority baseline
- **Cơ sở:** p̂=0,7565, n=5.687, Binomial kỳ vọng 4.302, std đếm 32,4. **Majority-baseline
  accuracy = 0,7565** → mốc mọi model phải vượt; và vì baseline đã cao nhờ mẫu lệch → đánh
  giá bằng **ROC-AUC/PR**.

### Bảng — Nhận diện phân phối lý thuyết
- **Cơ sở** (`deep_eda_distribution_id.csv`), quy tắc |skew|<0,5 **và** |kurt|<1:
  tuổi skew −0,084 nhưng **kurt −1,042** → không Normal (quá phẳng); GPA −0,580; đổi việc
  1,489; duration 1,890 → **cả 5 biến non-normal**. Shapiro p cực nhỏ ở n lớn là bình
  thường → quyết bằng moment, không bằng riêng p.

### Biểu đồ 9 — Ứng viên biến đổi
- **Loại:** barh nhóm (|skew| gốc vs sau biến đổi), vạch ngưỡng 0,5.
- **Ý nghĩa / phân tích gì:** phép biến đổi nào kéo lệch về gần 0 cho biến lệch phải.
- **Cơ sở kết luận** (`deep_eda_transform_candidates.csv`): duration 1,89→**log1p**
  −0,34; job_changes 1,49→**sqrt** 0,68; total_jobs 1,49→**log1p** 1,00; tuổi/GPA→**none**
  (cải thiện <0,1). Là **đề xuất tiền-mô-hình-hóa**; lưu ý các biến này vẫn bị loại vì
  leakage nên chỉ áp cho phân tích phụ.

---

## Phần G — Mục 7: Liên hệ feature × target

### Biểu đồ 10 — Lưới tỷ lệ có việc theo nhóm (11 ô)
- **Loại:** small-multiples barh, vạch nét đứt = TB chung 75,7%.
- **Ý nghĩa / phân tích gì:** mức **phân tách outcome** của từng biến phân loại.
- **Cơ sở kết luận:** khoảng cách thanh so với đường chung = cường độ. Lọc nhóm n≥20 cho
  ổn định. Nhấn mạnh: chênh lệch là **liên hệ trong mẫu**, không nhân quả.

### Bảng — Kiểm định thống kê (Welch t + χ²)
- **Cơ sở** (`yp2021_eda_stat_tests.csv`), p tăng dần:
  - **Tuổi** mạnh nhất: Welch t p≈3,9·10⁻⁷⁵ (TB 27,53 vs 26,15).
  - job_changes/total_jobs p≈8·10⁻⁸; duration p≈4·10⁻⁷.
  - **chứng chỉ** χ² p≈3,1·10⁻⁸; ngành 9,7·10⁻⁵; học vấn cha 1,7·10⁻³; học vấn
    4,5·10⁻³; mẹ 6,9·10⁻³; giới tính 1,5·10⁻².
  - **Không đạt 5%:** loại trường 0,097 · tài sản 0,234 · **GPA 0,060** · chuẩn bị việc
    0,764 — phần do cỡ mẫu hợp lệ nhỏ → cơ sở *không overclaim*.

### Biểu đồ 11 — Heatmap tương quan Pearson (gồm target 0/1)
- **Loại:** heatmap đối xứng (RdBu, −1..1) có ghi số.
- **Ý nghĩa / phân tích gì:** vừa đo liên hệ biến số↔target, vừa lộ **cặp tự tương quan**.
- **Cơ sở kết luận:** `total_jobs ≈ job_changes + 1` → tương quan ≈1,0 (point-biserial
  của chúng với target trùng tới 4 chữ số 0,0716) → cơ sở loại bớt 1 biến.

### Biểu đồ 12 — Xếp hạng liên hệ feature × outcome
- **Loại:** barh ngang, màu theo kind (numeric/categorical).
- **Ý nghĩa / phân tích gì:** quy mọi feature về **một thang cường độ chung** (|point-
  biserial r| hoặc Cramér's V) để xếp hạng.
- **Cơ sở kết luận** (`yp2021_feature_target_association.csv`): tuổi **0,241** ≫ ngành
  0,152 *(n=948, hẹp ~17% → thận trọng)* > duration 0,114 > vùng 0,079 > chứng chỉ 0,072
  > job count 0,072 > GPA 0,061 > loại trường 0,053 > cha 0,049 > mẹ 0,042 > học vấn 0,039
  > giới 0,029 > tài sản 0,019. Ngay biến mạnh nhất chỉ r≈0,24 → giải thích ROC-AUC ~0,68.

### Bảng — Subgroup ranking + Wilson CI + tầng độ chắc
- **Cơ sở:** mỗi subgroup kèm **Wilson CI 95%** + tầng 🔵/🟡/🟢. *Cơ sở phân tầng:* 🔵 =
  χ² ý nghĩa **và** CI không chứa tỷ lệ chung; 🟡 = CI chồng chung **hoặc** thuộc subset
  ~17%; 🟢 = không khác. VD Seoul 0,797 [0,772; 0,820] = 🔵; mọi mức ngành bị hạ 🟡 vì
  selection bias → chặn thổi phồng chênh lệch nhỏ thành "phát hiện".

### Biểu đồ 13 — Heatmap đa biến giữa predictor (Cramér's V)
- **Loại:** heatmap (YlOrRd, 0..1).
- **Ý nghĩa / phân tích gì:** liên hệ **giữa các predictor với nhau** (không phải target)
  → phát hiện đa cộng tuyến/trùng thông tin.
- **Cơ sở kết luận:** cặp V cao = trùng thông tin → gợi ý lược bớt khi model. Bổ trợ
  cảnh báo `total_jobs ≈ job_changes + 1` ở Biểu đồ 11.

---

## Phần H — Logic chọn / loại trường vào mô hình (trả lời trực tiếp)

Đây là phần đề bài hỏi: **với nhóm độ phủ cao, tại sao chọn trường này, bỏ trường kia?**
Câu trả lời: feature phải qua **BA CỬA** mới vào model lõi — **độ phủ cao *một mình
không đủ***.

**Ba cửa lọc (phải qua cả ba):**
1. **Cửa độ phủ (coverage):** phủ gần đầy mẫu pilot, *không* thuộc tiểu mô-đun ~17%.
2. **Cửa leakage:** đo *tại/trước* thời điểm quan sát việc làm wave 4 — không đo *sau*.
3. **Cửa chệch lựa chọn:** thuộc nhóm phủ đầy, không phải subset "vừa tốt nghiệp đợt này".

**Bảng quyết định đầy đủ** (`yp2021_feature_catalog.csv`, coverage thực đo):

| Feature | Coverage | Cửa leakage | Cửa chọn lọc | **Quyết định** |
|---|---:|---|---|---|
| `age_w04` (số) | 100% | ✅ trước/tại w4 | ✅ phủ đầy | **CORE** |
| `gender_label` | 100% | ✅ | ✅ | **CORE** |
| `region_group_w04_label` | 100% | ✅ | ✅ | **CORE** |
| `education_level_w04_label` | 100% | ✅ | ✅ | **CORE** |
| `certificate_earned_label` | 100% | ✅ | ✅ | **CORE** |
| `parent_asset_bracket_label` | 99,7% | ✅ | ✅ | **CORE** |
| `mother_education_label` | 98,6% | ✅ | ✅ | **CORE** |
| `father_education_label` | 95,6% | ✅ | ✅ | **CORE** |
| `total_jobs` | **86,7%** (cao!) | ❌ đo **sau** việc làm | ✅ | **LOẠI — leakage** |
| `job_changes` | **86,7%** (cao!) | ❌ đo **sau** | ✅ | **LOẠI — leakage** |
| `avg_job_duration_months` | 30,2% | ❌ đo **sau** | ✅ | **LOẠI — leakage** |
| `graduated_major_field_label` | 16,7% | ✅ | ❌ subset ~17% | **NARROW** (sensitivity) |
| `graduate_university_type_label` | 16,6% | ✅ | ❌ | **NARROW** |
| `graduation_gpa_yp100` | 16,6% | ✅ | ❌ | **NARROW** |
| `job_prep_gpa_spec_label` | 13,6% | ✅ | ❌ | **NARROW** |

**Kết luận trực tiếp cho câu hỏi "độ phủ cao nhưng vẫn bỏ":**
- `total_jobs` và `job_changes` **phủ tới 86,7%** — cao hơn cả `father_education` (95,6%
  vào core). Nhưng chúng **đếm việc làm *sau* khi tình trạng việc đã quan sát** → là
  **leakage**: dùng làm feature sẽ "biết trước đáp án", điểm số đẹp giả tạo và sụp khi
  triển khai thật. Vì vậy **loại hẳn khỏi cả core lẫn narrow** (`LEAKAGE_COLUMNS` trong
  `dataset.py`). → *Độ phủ cao không cứu được biến bị rò rỉ.*
- Ngược lại, nhóm ngành/GPA/loại trường **không leakage** nhưng **phủ chỉ ~17%** và là
  **subset chọn lọc** (mô-đun "vừa tốt nghiệp đợt này", tỷ lệ có việc 51,8% ≠ 75,6% toàn
  mẫu). Impute như MCAR sẽ tiêm chệch lựa chọn → chỉ cho vào model `--narrow-scope` kèm
  **cờ chỉ báo khuyết** để đọc thận trọng, không dùng cho kết luận chính.

**Vậy 8 feature vào CORE:** `age_w04` + 7 categorical (gender, region, education_level,
certificate, father/mother_education, parent_asset). **3 loại vì leakage. 4 chỉ vào
narrow.** Catalog sinh từ chính các tuple trong `dataset.py` nên **không thể lệch** khỏi
model thực dùng.

**Tiền xử lý feature core (trong sklearn `Pipeline`, fit chỉ trên fold train):**
- Numeric (`age_w04`, và `gpa` ở narrow): `SimpleImputer(median)` → `StandardScaler`.
- Categorical: `SimpleImputer(constant='missing')` → `OneHotEncoder(handle_unknown=
  'ignore')`. *Lý do `handle_unknown='ignore'`:* hạng mục chỉ thấy ở test không làm vỡ
  pipeline, không cần nhìn trước test để fit encoder → chống leakage.

---

## Phần I — Mục 9: Modeling & đánh giá nâng cao

### Bảng — Modeling-ready (9.1)
- **Cơ sở:** 4 kiểm tra hợp đồng có `assert` *trước* khi fit: (1) grain 1 dòng/người;
  (2) target nhị phân 0/1 đã tách X; (3) không leakage trong X; (4) missingness <100%/cột.
  Fail → dừng, đảm bảo metric sau đáng tin.

### Bảng metric 4 mô hình (9.2)
- **Cơ sở** (`yp2021_model_metrics.csv`):

  | Mô hình | Acc | ROC-AUC | Avg.Prec | Brier | Recall |
  |---|---:|---:|---:|---:|---:|
  | baseline most-frequent | 0,757 | **0,500** | 0,757 | 0,243 | 1,000 |
  | logreg L2 | 0,760 | 0,679 | 0,855 | **0,170** | 0,983 |
  | logreg L1 | 0,761 | 0,679 | 0,855 | 0,170 | 0,984 |
  | random forest | 0,656 | **0,681** | 0,853 | 0,220 | 0,668 |

  *Đọc:* baseline accuracy 0,757 ≈ logistic **nhưng** AUC=0,5 → không phân biệt được ai
  thất nghiệp. Logistic vượt rõ AUC/AP/Brier. RF AUC nhỉnh nhưng đổi recall lấy precision.

### Mục 9.3 — Cầu nối toán học (markdown)
- **Cơ sở:** sigmoid, log-odds=z, cross-entropy (NLL Bernoulli), gradient
  (1/n)Σ(p̂−y)x, baseline-theo-loss (MSE→mean, MAE→median, log-loss→prevalence) → giải
  thích vì sao baseline = most-frequent và AUC=0,5.

### Biểu đồ 14 — Bốn ma trận nhầm lẫn
- **Loại:** 4 imshow 2×2 (Blues) có annot.
- **Ý nghĩa / phân tích gì:** mở bung metric thành 4 ô quyết định, soi khả năng bắt **lớp
  thiểu số** (thất nghiệp).
- **Cơ sở kết luận** (tn,fp,fn,tp): baseline (0,277,0,861) → đoán "có việc" cho **tất
  cả**, bắt **0/277** thất nghiệp. L2 (19,258,15,846) → bắt 19/277. RF (171,106,286,575)
  → bắt **171/277** nhưng bỏ sót nhiều người có việc → accuracy thấp hơn. *Thông điệp:*
  giá trị nằm ở nhận diện lớp thiểu số, accuracy đơn lẻ che giấu.

### Biểu đồ 15 — ROC + Precision-Recall (2 panel)
- **Loại:** 2 line chart, kèm đường tham chiếu.
- **Ý nghĩa / phân tích gì:** năng lực phân tách ở **mọi ngưỡng**; PR ưu tiên lớp dương.
- **Cơ sở kết luận:** 3 model thật vượt rõ tham chiếu (ROC chéo 0,5; PR prevalence 0,757);
  AP≈0,855 ≫ 0,757 → tín hiệu thật. PR phù hợp hơn vì lớp lệch.

### Biểu đồ 16 — Calibration + Threshold tuning (2 panel)
- **Loại:** 2 line chart.
- **Ý nghĩa / phân tích gì:** (trái) xác suất dự đoán có khớp tần suất quan sát không;
  (phải) F1 thay đổi ra sao khi dời ngưỡng khỏi 0,5.
- **Cơ sở kết luận:** Brier L2 0,170 ≪ baseline 0,243 → xác suất logistic hiệu chỉnh tốt,
  đáng tin khi ra quyết định. Threshold (`yp2021_model_thresholds.csv`): dời ngưỡng đánh
  đổi precision–recall → chỉnh khi chi phí 2 loại sai khác nhau.

### Biểu đồ 17 — Hiệu năng L2 theo nhóm (subgroup fairness)
- **Loại:** barh nhóm theo giới/vùng/học vấn, vạch AUC tổng thể.
- **Ý nghĩa / phân tích gì:** model có đồng đều giữa nhóm không (công bằng).
- **Cơ sở kết luận** (`yp2021_model_subgroups.csv`, n≥30): gap ROC-AUC giữa nhóm **nhỏ** →
  tương đối đồng đều, chưa thấy bất công nghiêm trọng. Vẫn là liên hệ trong pilot. Ngưỡng
  n≥30 để mỗi ước lượng đủ ổn định.

### Biểu đồ 18 — Top 15 hệ số log-odds (L2)
- **Loại:** barh phân kỳ (xanh tăng / cam giảm).
- **Ý nghĩa / phân tích gì:** mỗi đặc trưng làm tăng/giảm *log tỉ số odds* có việc.
- **Cơ sở kết luận:** dấu + độ lớn **nhất quán với EDA** (tuổi/chứng chỉ/vùng/nền tảng
  cha mẹ nổi bật) → khép kín mạch khám phá → mô hình, tăng độ tin diễn giải.

### Bảng — Đánh đổi bias–variance (9.6)
- **Cơ sở** (`metrics_frame`): baseline CV train=test=0,5 → **bias cao** (underfit). L2:
  CV train 0,694 vs CV test 0,674 → **gap hẹp ≈0,02** → điều chuẩn kiểm soát variance,
  không overfit. RF: CV train 0,754 vs CV test 0,671 → **gap rộng ≈0,08** (variance cao)
  nhưng test AUC tương đương L2. *Kết luận:* RF không thắng dù phức tạp hơn → **trần
  thông tin của tập feature** (sau loại leakage) mới là giới hạn, không phải dạng model.

---

## Phần J — Tổng hợp cơ sở kết luận

| Kết luận chính | Cơ sở số liệu/kiểm định |
|---|---|
| ~¾ người tốt nghiệp có việc ở w4 | p̂=0,7566; CI 95% [0,745; 0,768] trùng 3 phương pháp |
| Tín hiệu phân loại thật nhưng yếu | Bootstrap AUC CI [0,641; 0,715] loại 0,5; AUC≈0,68 |
| Model ổn định | Split std 0,017 (AUC), 0,005 (acc) |
| Liên hệ mạnh nhất: tuổi, chứng chỉ, vùng, nền tảng cha mẹ | r/Cramér's V + Welch t/χ² (p<0,05); log-odds L2 nhất quán |
| Không overclaim ngành/GPA/loại trường | n≤948, p>0,05 hoặc tier 🟡 selection bias |
| 8 feature vào core, 3 loại leakage, 4 narrow | catalog 3 cửa lọc (coverage/leakage/chọn lọc) |
| Logistic ổn hơn RF dù đơn giản | gap train–CV 0,02 vs 0,08, test AUC tương đương |

**Giới hạn đọc kèm:** lọc học vấn → chệch bao phủ; attrition 21% → chệch không phản hồi;
subset ~17% → chệch lựa chọn (🟡); job-history hồi tưởng + GPA chuẩn hoá riêng → chệch đo
lường. Mọi kết luận: **liên hệ/dự đoán, không nhân quả**, chỉ trong **pilot wave 4 sau
tốt nghiệp**, không gộp 3 dataset.
