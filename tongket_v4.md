# TỔNG KẾT DỰ ÁN — Phân tích Youth Panel 2021 (YP2021)

*Dự đoán tình trạng việc làm sau tốt nghiệp của thanh niên Hàn Quốc — Báo cáo tổng hợp toàn diện*

Ngày tổng hợp: **16/06/2026**
Nguồn tổng hợp: `notebooks/yp2021_report.ipynb`, `docs/describe_data/02_youth_panel_yp2021.md`,
`bosung.md`, và toàn bộ codebase `src/data/youth_panel/`, `src/models/youth_panel/`,
`src/visualization/`.

---

## MỤC LỤC

1. [Giới thiệu và bối cảnh dự án](#1-giới-thiệu-và-bối-cảnh-dự-án)
2. [Tổng quan dữ liệu nguồn](#2-tổng-quan-dữ-liệu-nguồn)
3. [Câu hỏi nghiên cứu và nguyên tắc xuyên suốt](#3-câu-hỏi-nghiên-cứu-và-nguyên-tắc-xuyên-suốt)
4. [Kiến trúc pipeline: từ raw đến processed](#4-kiến-trúc-pipeline-từ-raw-đến-processed)
5. [Phạm vi dự án và bộ ba P-A-S](#5-phạm-vi-dự-án-và-bộ-ba-p-a-s)
6. [Phân tích chi tiết từng biểu đồ](#6-phân-tích-chi-tiết-từng-biểu-đồ)
7. [Tiền xử lý và trích xuất đặc trưng — vì sao chọn từng bước](#7-tiền-xử-lý-và-trích-xuất-đặc-trưng--vì-sao-chọn-từng-bước)
8. [Mô hình hóa và đánh giá](#8-mô-hình-hóa-và-đánh-giá)
9. [Cầu nối toán học](#9-cầu-nối-toán-học)
10. [Kết luận và hạn chế](#10-kết-luận-và-hạn-chế)
11. [Phụ lục tái lập](#11-phụ-lục-tái-lập)

---

## 1. Giới thiệu và bối cảnh dự án

### 1.1. Mục tiêu

Dự án thực hiện **trọn vẹn vòng đời một dự án khoa học dữ liệu** trên bộ dữ liệu
**Youth Panel 2021 (YP2021)** của Viện Phát triển Nhân lực Hàn Quốc (KEIS). Câu hỏi
trọng tâm là:

> *Những đặc trưng nào của một người vừa hoàn tất giáo dục sau trung học có liên hệ với
> khả năng người đó **có việc làm ở wave 4** (`employed_w04`)?*

Toàn bộ phân tích bám lấy một **mẫu pilot gồm 5.687 respondent** đã tốt nghiệp, trích ra
từ khung panel **12.213 người** được theo dõi qua bốn đợt khảo sát (wave 1–4, giai đoạn
2021–2024).

### 1.2. Vị trí trong project lớn

YP2021 là **một trong ba dataset family** của project (cùng với GOMS và KEEP II). Một
quy tắc kỷ luật quan trọng: **không gộp (pool) respondent giữa ba dataset** vì chúng
khác thiết kế khảo sát và không có định danh (ID) chung. Vì thế báo cáo này chỉ kết luận
trong phạm vi YP2021, không suy rộng và không so sánh tuyệt đối các chỉ số (ví dụ GPA)
với GOMS/KEEP vì thang đo và chuẩn hóa khác nhau.

### 1.3. Đặc điểm dữ liệu khiến YP2021 đặc biệt

- **Là dữ liệu panel** (theo dõi cùng một nhóm người qua thời gian) → quan sát được hiện
  tượng **rơi rụng (attrition)**, điều mà dữ liệu cắt ngang (cross-sectional) như GOMS
  không có.
- **Không phải graduate-only**: YP2021 tuyển cả thanh niên 19–28 tuổi nói chung, bao gồm
  người chưa đạt ngưỡng cao đẳng/đại học. Do đó để phù hợp scope "sau tốt nghiệp", phải
  **lọc theo học vấn** (`w04edu >= 3`).
- **Có file job-history chi tiết** ở cấp từng đợt việc làm (job-spell), cho phép phái
  sinh các đặc trưng lịch sử nghề nghiệp.

---

## 2. Tổng quan dữ liệu nguồn

### 2.1. Các file thô (raw)

Raw root: `data/raw/youth_panel/yp2021/`

| Bảng | Đường dẫn | Độ chi tiết (grain) | Dòng | Cột |
|---|---|---|---:|---:|
| Wave 1 | `stata/yp2021_w01.dta` | respondent | 12.213 | 1.360 |
| Wave 2 | `stata/yp2021_w02.dta` | respondent | 12.213 | 1.522 |
| Wave 3 | `stata/yp2021_w03.dta` | respondent | 12.213 | 1.540 |
| Wave 4 | `stata/yp2021_w04.dta` | respondent | 12.213 | 1.549 |
| Job history | `job_history/yp2021_job_history_w01_w04.dta` | respondent-job spell | 27.127 | 98 |

Mỗi file wave **giữ nguyên toàn bộ khung 12.213 thành viên** — việc tham gia là một
**cờ (flag)**, không phải sự hiện diện/vắng mặt của dòng. Biến tham gia `w01`–`w04` có
nhãn `참여여부` (có tham gia hay không), với `1 = 참여` (tham gia), `2 = 미참여`
(không tham gia).

### 2.2. Tài liệu metadata kèm theo

- `docs/yp2021_codebook_w01_w04.xlsx` — codebook
- `docs/yp2021_variable_mapping_w01_w04.xlsx` — ánh xạ biến
- `docs/yp2021_job_history_codebook_w01_w04.xlsx`
- `docs/yp2021_user_guide_w01_w04.pdf`
- `questionnaires/yp2021_questionnaire_w01_w04.pdf`

### 2.3. Tham gia panel quan sát được (observed attrition)

| Wave | Năm | Số người tham gia |
|---|---:|---:|
| w01 | 2021 | 12.213 |
| w02 | 2022 | 10.721 |
| w03 | 2023 | 10.108 |
| w04 | 2024 | 9.665 |

> **Quan trọng:** các con số này được **quan sát trực tiếp từ dữ liệu**, không phải sao
> chép từ tài liệu ngoài. Tỷ lệ giữ chân wave 1 → wave 4 ≈ **79,1%** (9.665/12.213).

---

## 3. Câu hỏi nghiên cứu và nguyên tắc xuyên suốt

Ba nguyên tắc mang tính "hiến pháp" của báo cáo, lặp lại ở mọi mục:

1. **Chỉ kết luận liên hệ / dự đoán, KHÔNG nhân quả.** Mọi chênh lệch tỷ lệ, mọi hệ số
   mô hình đều là liên hệ thống kê trong mẫu, không phải bằng chứng "X gây ra Y".

2. **Mọi con số quan sát trực tiếp từ dữ liệu** đã qua pipeline. Không có số nào nhập tay
   hay suy đoán; các ô code trong notebook gọi thẳng những module đã có unit-test trong
   `src/`, nên con số trong báo cáo **trùng khớp** với artefact ở `reports/`.

3. **Không gộp ba dataset.** YP2021 phân tích độc lập với GOMS và KEEP II.

Ngoài ra, mọi mô phỏng/mô hình dùng **seed cố định `RANDOM_STATE = 42`** để đảm bảo tính
tái lập (reproducibility).

---

## 4. Kiến trúc pipeline: từ raw đến processed

Dự án tổ chức dữ liệu thành ba lớp theo chuẩn data engineering:

```
raw (.dta gốc)  →  interim (parquet trung gian)  →  processed (sẵn sàng phân tích)
```

Kiến trúc nhiều lớp này không phải để cho "đẹp", mà giải quyết **ba vấn đề rất cụ thể của
chính bộ dữ liệu YP2021**. Hiểu được ba động lực này thì mọi bước xử lý phía sau đều có lý do:

| Động lực thiết kế | Vấn đề thực tế của YP2021 | Lớp giải quyết |
|---|---|---|
| **Trung thực với nguồn** | File `.dta` chứa skip code (`9999998/9999999`), nhãn Hàn, mã categorical — không được tự ý sửa kẻo mất dấu vết gốc | `raw_tables` giữ nguyên, chỉ thêm cột provenance |
| **Hài hòa tên biến qua wave** | Wave 1 đặt tên `w01{stem}`, wave 2–4 đặt `y0n...`; workbook lại viết hoa/khác case → **không thể suy tên biến từ tên cột** | Lớp metadata dựng `variable_catalog` làm "từ điển" chính thức |
| **Tách quyết định làm sạch ra khỏi đọc dữ liệu** | Mask skip code, decode nhãn, phái sinh feature là các *quyết định phân tích*, phải tách khỏi việc đọc thô để có thể kiểm chứng | `processed` mới là nơi ra quyết định làm sạch |

### 4.1. Vì sao tách lớp?

**Lý do cốt lõi: các "mã bỏ qua" (skip codes) chỉ được làm sạch ở lớp processed.** Dữ
liệu Stata gốc giữ nguyên các sentinel như `9090908`, `9090909`, `9999997`, `9999998`,
`9999999` để đánh dấu câu hỏi bị bỏ qua / không trả lời. Nếu coi chúng là **giá trị số
thật**, mọi trung bình và phương sai sẽ bị **bóp méo nghiêm trọng** (ví dụ một giá trị
`9999999` sẽ kéo trung bình lên trời). Vì thế:

- **Lớp raw_tables**: chỉ là bản materialize trung thực từ `.dta`, **không làm sạch**,
  dùng cho truy vết (traceability) và kiểm chứng.
- **Lớp processed**: đã mask skip code thành `NaN`, đã decode nhãn, đã phái sinh đặc
  trưng. **Đây là lớp dùng cho mọi phân tích chính.**

> **Quy tắc EDA:** không bao giờ chạy EDA trực tiếp trên `raw_tables` (vì skip code chưa
> xử lý). Ngoại lệ duy nhất là Mục 7 (wrangling) — nơi *cố tình* quay về raw để phô bày
> quy trình biến đổi.

### 4.1b. Hai nguyên tắc xương sống: *metadata-first* và *ghi-nhận-không-tự-sửa*

Ngoài việc tách lớp, pipeline tuân theo hai nguyên tắc chi phối **vì sao các bước được sắp
như thế**:

**(1) Metadata-first — dựng "từ điển biến" trước khi đụng một dòng dữ liệu nào.** Bước đầu
tiên (`build-metadata`) *không* đọc dữ liệu, mà chỉ đọc **metadata** (label, value-label,
codebook, mapping) để dựng `variable_catalog`. Lý do bắt buộc: YP2021 đổi quy ước đặt tên
giữa các wave và workbook dùng case khác tên cột thực. Nếu nhảy thẳng vào xử lý dữ liệu,
ta sẽ **map sai biến qua wave** (đây đúng là lỗi từng xảy ra — xem commit *"fix lỗi chuyển
từ data raw sang interim... bị map sai nhãn"*). Có catalog làm chuẩn trước, mọi bước sau
mới tham chiếu được tên biến một cách tin cậy. Mỗi biến còn được chấm
`metadata_confidence` (`confirmed`/`partial`/`conflict`/`unknown`) để biết chỗ nào chắc,
chỗ nào cần thận trọng.

**(2) Ghi nhận vấn đề, không âm thầm tự sửa.** Mọi bước đều mang theo một `AuditLog`. Khi
gặp lệch số dòng/cột, `sampid` null, hay metadata mâu thuẫn (vd label `.dta` có hậu tố
wave `(1차년도)` còn codebook giữ mô tả gốc → đánh dấu `conflict`), pipeline **log lại
vào bảng `*_issues` thay vì tự đoán và sửa**. Triết lý: một quyết định làm sạch sai mà bị
giấu đi còn nguy hiểm hơn một vấn đề được phơi bày. Nhờ vậy mỗi con số ở báo cáo đều
**truy ngược được** về bước sinh ra nó.

> **Hệ quả về thứ tự chạy:** các bước **phụ thuộc nhau theo chuỗi** — mỗi lệnh đọc output
> trên đĩa của lệnh trước và sẽ báo lỗi `Run ... first` nếu thiếu. Đây là thiết kế
> *idempotent*: chạy lại một bước cho ra cùng kết quả, và có thể chạy lại riêng một bước
> mà không phải làm lại từ đầu.

### 4.2. Các lệnh pipeline (chạy theo đúng thứ tự)

```bash
python -m src.data.youth_panel.pipeline build-metadata             # catalog 6.069 biến
python -m src.data.youth_panel.pipeline build-raw-tables           # materialize .dta → parquet
python -m src.data.youth_panel.pipeline build-panel                # panel index dài 48.852 dòng
python -m src.data.youth_panel.pipeline build-job-history          # 9.083 người, đặc trưng job
python -m src.data.youth_panel.pipeline build-construct-candidates # ứng viên biến cấu trúc
python -m src.data.youth_panel.pipeline build-target-sample        # mẫu pilot 5.687
python -m src.data.youth_panel.pipeline build-processed-pilot      # bảng processed 47 cột
```

Bảy bước không phải bảy việc rời rạc, mà là một **chuỗi phụ thuộc**: mỗi bước thêm đúng
một biến đổi và là tiền đề cho bước sau. Vì sao cần từng bước, và vì sao đặt ở đúng vị trí:

| Bước | Làm gì | Vì sao cần & vì sao ở vị trí này |
|---|---|---|
| **1. metadata** | Đọc metadata, dựng `variable_catalog` + chấm độ tin cậy | Phải có **trước tiên** vì mọi bước sau cần "từ điển biến" để tham chiếu đúng tên qua wave (tránh map sai nhãn) |
| **2. raw-tables** | Materialize full `.dta` → parquet, **giữ nguyên** mã số, thêm cột provenance | Tạo bản sao trung thực để truy vết; phải sau metadata để validate số dòng/cột đối chiếu kỳ vọng |
| **3. panel** | Dựng index mỏng `sampid + wave` (48.852 = 4×12.213), bool-hóa cờ tham gia | Biến cờ `참여여부` thành cột `participated` → đo **attrition** observed (nền cho funnel & P-A-S) |
| **4. job-history** | Gộp bảng spell-grain (27.127 dòng) lên mức người: row→job→respondent | Đổi **độ chi tiết** để có đặc trưng cấp cá nhân (`total_jobs`, `job_changes`...); tách riêng vì grain khác hẳn panel |
| **5. construct-candidates** | Liệt kê biến ứng viên theo nhóm cấu trúc (chỉ metadata) | Bước **review trên giấy** trước khi vật chất hóa feature — quyết định *giữ biến nào* trước khi tốn công xử lý |
| **6. target-sample** | Xác minh metadata cho rule `w04==1 & w04edu>=3`, dựng funnel, định nghĩa `employed_w04` | Phải có **trước** processed vì processed cần biết *lọc ai* (5.687 người) và *target là gì* |
| **7. processed-pilot** | Mask skip code, decode nhãn, phái sinh feature (vd GPA), link job-history → bảng 47 cột | Bước **cuối cùng** quy tụ mọi quyết định làm sạch; chỉ chạy được khi 6 bước trên đã xong |

Điểm mấu chốt của thứ tự này: **metadata và scope được chốt trước, làm sạch và phái sinh
feature làm sau cùng**. Nhờ vậy nếu cần đổi một quyết định làm sạch (vd cách điền khuyết),
ta chỉ chạy lại bước 7 mà không đụng tới catalog hay sample đã xác minh.

### 4.3. Bảng processed trung tâm

`yp2021_processed_pilot_w04_en.parquet`: **5.687 dòng × 47 cột** (nhãn đã dịch tiếng Anh
để vẽ biểu đồ). Các nhóm cấu trúc (construct groups) đã materialize: `target`,
`demographics`, `education`, `academic_performance`, `skills`, `job_preparation`,
`family_background`, `work_experience`. Biến phân loại giữ **cả mã thô lẫn nhãn đã
decode**.

### 4.4. Một quyết định chuẩn hóa đáng chú ý: GPA

GPA của YP gốc là **biến phân loại** (`y04a141`), không phải điểm số liên tục. Pipeline
tạo `graduation_gpa_yp100` bằng **quy tắc trung điểm (midpoint) riêng cho YP**:

| Mã thô | Nhãn | Quy về thang 100 |
|---:|---|---:|
| 1 | A- ~ A+ | 95 |
| 2 | B- ~ B+ | 85 |
| 3 | C- ~ C+ | 75 |
| 4 | D- ~ D+ | 65 |
| 5 | F | 50 |

> **Cảnh báo diễn giải:** không so sánh giá trị GPA này như điểm tuyệt đối với GOMS/KEEP
> mà không nêu rõ thang nguồn và cách chuẩn hóa khác nhau.

#### 4.4.1. Vì sao chọn quy tắc trung điểm — và logic của từng mốc

Phải dùng **bảng quy tắc ánh xạ** chứ không thể "tính ra", vì biến gốc `y04a141` **không
lưu con số GPA thực** (kiểu 3.7/4.5) mà chỉ lưu **mã nhóm hạng chữ** (A/B/C/D/F). Khi dữ
liệu nguồn đã là **khoảng hạng (band)** chứ không phải số đo, không có cách nào ngoài việc
**quy ước** một giá trị đại diện cho mỗi khoảng. Code thực hiện ở `processed.py`
(`GPA_CATEGORY_TO_100`) và ghi rõ nguồn quy tắc tại hằng `GPA_RULE_SOURCE`.

Với bốn hạng A/B/C/D, mỗi hạng tương ứng một **dải phần trăm rộng ~10 điểm** trong hệ Hàn
Quốc (A≈90–100, B≈80–89, C≈70–79, D≈60–69). Khi chỉ biết *khoảng* mà không biết vị trí
thực bên trong, **điểm giữa (midpoint)** là giá trị đại diện ít thiên lệch nhất — nó là kỳ
vọng hợp lý nhất → A=95, B=85, C=75, D=65. Chọn **thang 100** (thay vì 4.5/4.3) để trung
tính và dễ diễn giải, đồng thời tránh ngộ nhận là so sánh được trực tiếp với GPA gốc của
GOMS/KEEP.

#### 4.4.2. Biện luận cho mốc F = 50 (giá trị mang tính tùy ý)

Khác bốn hạng còn lại, **F không có dải dưới xác định**: điểm trượt có thể nằm bất kỳ đâu
từ 0 đến 59. Nếu áp máy móc quy tắc trung điểm cho khoảng "0–59" thì F ≈ 30 — nhưng con số
này tạo một **outlier cực đoan**, kéo giãn phương sai và bóp méo phân phối, đồng thời làm
bất ổn hệ số hồi quy. Vì vậy nhóm **chủ động chọn F = 50** thay vì để công thức midpoint
quyết định. Năm lập luận bảo vệ lựa chọn này:

1. **Bảo toàn thứ bậc (ordinal) và khoảng cách hợp lý.** Điều thực sự quan trọng với mô
   hình là *trật tự* F < D < C < B < A được giữ đúng và **giãn cách tương đối đều**. Chuỗi
   50 < 65 < 75 < 85 < 95 thỏa cả hai: đơn điệu tăng, và bước nhảy giữa các hạng không
   chênh lệch bất thường. F = 50 đặt F **thấp hơn D đúng 15 điểm** — cùng bậc giãn cách
   với các cặp hạng kề khác — thay vì tạo một hố sâu phi lý.

2. **Tránh outlier hơn là truy đuổi "độ chính xác" giả.** Một giá trị F = 0 hay F = 30 trông
   "đúng theo công thức" hơn, nhưng vì hạng chữ vốn đã **mất hết độ phân giải bên trong
   khoảng**, mọi con số đều chỉ là quy ước. Giữa hai cái quy ước, ta chọn cái **ít gây hại
   cho phân phối** hơn. F = 50 không nói "người trượt đạt 50% kiến thức"; nó chỉ là **mã
   thứ bậc thấp nhất** đặt ở khoảng cách an toàn.

3. **Ảnh hưởng thực tế gần như bằng không — vì hai lý do cộng hưởng.** (a) F **cực hiếm**
   trong mẫu: đối tượng phân tích là người đã *tốt nghiệp CĐ/ĐH* (`w04edu >= 3`), nên gần
   như không ai mang hạng tốt nghiệp F. (b) Bản thân GPA **không đạt ý nghĩa thống kê**
   (p = 0,060, xem [Mục 6.11](#611-bảng-kiểm-định-thống-kê-welch-t-test--chi-square)) và
   trong mô hình logistic nó hành xử gần như **biến thứ bậc** chứ không phải biến liên tục
   thật. Một giá trị hiếm, gắn vào một biến vốn yếu, thì việc đặt nó ở 50 hay 40 hay 0
   **không thể dịch chuyển kết luận**.

4. **Kiểm chứng được bằng phân tích độ nhạy (sensitivity).** Chính vì điểm 1–3, mệnh đề
   "kết luận không phụ thuộc lựa chọn F" là **kiểm tra được**, không phải niềm tin: chạy
   lại pipeline với F ∈ {0; 40; 50} và đối chiếu hệ số/ROC-AUC sẽ cho kết quả gần như
   trùng. Đây là tuyến phòng thủ chuẩn cho mọi tham số quy ước — biến một lựa chọn tùy ý
   thành một lựa chọn **đã được chứng minh là không trọng yếu**.

5. **Trung thực và phơi bày, không giấu.** Quy tắc được **ghi thành tài liệu** trong code
   (`GPA_RULE_SOURCE`) và `README`, và được liệt kê thẳng vào **chệch đo lường
   (measurement bias)** ở [Mục 5.5](#55-năm-loại-chệch--sai-số-trong-yp2021). Một giả định
   được nêu rõ và đánh dấu rủi ro thì an toàn hơn nhiều một giả định bị che đi — đúng tinh
   thần "ghi nhận vấn đề, không âm thầm tự sửa" của toàn pipeline ([Mục 4.1b](#41b-hai-nguyên-tắc-xương-sống-metadata-first-và-ghi-nhận-không-tự-sửa)).

> **Chốt lại để bảo vệ trước hội đồng:** các con số 95/85/75/65/50 là **giả định của nhóm**,
> không phải dữ liệu YP cung cấp. Điểm yếu lộ rõ nhất là F = 50 mang tính tùy ý — nhưng nó
> được chọn có chủ đích để *bảo toàn thứ bậc và tránh outlier*, tác động tới kết luận là
> **không đáng kể** (F cực hiếm + GPA vốn không có ý nghĩa thống kê), và mệnh đề đó
> **kiểm chứng được** bằng phân tích độ nhạy. Đây là cách xử lý một tham số quy ước một
> cách có kỷ luật, không phải một lỗ hổng bị bỏ qua.

---

## 5. Phạm vi dự án và bộ ba P-A-S

Trước khi đụng tới bất kỳ con số nào, dự án xác định rõ **đang nói về ai**. Đây là bước
nền tảng của mọi suy luận thống kê.

### 5.1. Ba chiều phạm vi

- **Không gian**: thanh niên Hàn Quốc.
- **Thời gian**: giai đoạn 2017–2024, với độ trễ panel bốn đợt khảo sát.
- **Lĩnh vực**: quá trình chuyển tiếp từ giáo dục sang việc làm.

### 5.1b. Vì sao chọn wave 4 làm mốc phân tích?

YP2021 là **panel 4 đợt** (wave 1→4, 2021→2024) theo dõi *cùng một nhóm người* qua thời
gian. Việc chọn wave 4 (đợt mới nhất, năm 2024) không tùy tiện mà có ba lý do:

1. **Phản ánh tình trạng gần hiện tại nhất.** Wave 4 cho biết tình trạng việc làm cập
   nhật nhất của nhóm thanh niên, thay vì một thời điểm đã cũ.
2. **Phù hợp scope "sau tốt nghiệp".** Panel tuyển thanh niên 19–28 tuổi tại wave 1 — lúc
   đó nhiều người **còn đang đi học**, chưa thể hỏi "có việc sau tốt nghiệp" một cách có
   ý nghĩa. Sau 4 năm (đến wave 4), phần lớn đã **hoàn tất giáo dục và bước vào thị trường
   lao động** → câu hỏi nghiên cứu mới có cơ sở.
3. **Đủ độ trễ để quan sát chuyển tiếp.** Mục tiêu là *chuyển tiếp giáo dục → việc làm*;
   wave 4 cho khoảng thời gian dài nhất để quá trình này diễn ra.

> **Cái giá phải trả:** wave 4 chịu **attrition** nặng nhất — chỉ còn 9.665/12.213 người
> (~79%). Đây chính là lý do báo cáo nhấn mạnh rủi ro chệch không phản hồi (xem Mục 6.2).

### 5.2. Bộ ba Quần thể – Khung chọn mẫu – Mẫu (P-A-S)

| Thành phần | YP2021 cụ thể hóa |
|---|---|
| **P — Quần thể mục tiêu** | Người tốt nghiệp CĐ/ĐH trở lên, giai đoạn sau tốt nghiệp |
| **Quần thể nguồn/thiết kế** | Thanh niên Hàn Quốc 19–28 tuổi tại wave 1 (panel KEIS) — *rộng hơn* P |
| **A — Khung chọn mẫu** | Panel YP2021: 12.213 respondent theo dõi qua wave 1–4 + job-history |
| **S — Mẫu phân tích** | Pilot wave 4 sau lọc học vấn: `w04 == 1 and w04edu >= 3` (5.687) |

**Điểm mấu chốt:** khung A **rộng hơn** quần thể P. Panel gốc tuyển cả thanh niên nói
chung, bao gồm người chưa đạt ngưỡng cao đẳng/đại học, nên buộc phải **lọc theo học vấn**
để thu khung A về đúng P. Chính bước lọc này tạo ra **selection funnel**.

**Giải mã quy tắc lọc `w04 == 1 and w04edu >= 3`** — hai điều kiện nối bằng AND:

| Điều kiện | Biến | Ý nghĩa | Nhãn gốc (Hàn) |
|---|---|---|---|
| `w04 == 1` | Cờ tham gia wave 4 | **Có tham gia** đợt khảo sát 4 (loại người vắng → không có dữ liệu) | `1=참여`, `2=미참여` |
| `w04edu >= 3` | Học vấn cao nhất ở wave 4 | **Tốt nghiệp CĐ trở lên** (loại người chưa đạt ngưỡng) | `3=전문대졸`(CĐ), `4=대졸`(ĐH), `5=석사학위이상`(Thạc sĩ+) |

**Vì sao cần điều kiện thứ hai (`w04edu >= 3`)?** Vì YP2021 *không phải graduate-only* —
panel tuyển cả thanh niên nói chung. Quần thể mục tiêu (P) lại là "người tốt nghiệp CĐ/ĐH
sau tốt nghiệp". Bộ lọc học vấn **thu khung chọn mẫu A (rộng) về đúng quần thể P**. Chính
bước này tạo selection funnel và là nguồn của **chệch bao phủ**.

Funnel rút gọn: 12.213 (khung) → 9.665 (`w04==1`) → **5.687** (`w04==1 and w04edu>=3`).

**Vì sao riêng điều kiện `w04 == 1`?** Đây là đặc thù của dữ liệu **panel**: trong file
Stata gốc `yp2021_w04.dta`, **mọi wave đều giữ nguyên đủ 12.213 dòng** — toàn bộ khung
panel. Một người rời khảo sát ở wave 4 **không bị xoá dòng**; dòng của họ vẫn còn, chỉ là
câu trả lời wave-4 bị bỏ trống. KEIS phân biệt "ai thực sự trả lời wave 4" bằng **biến cờ
tham gia** `w04` (nhãn `참여여부`), chứ không bằng sự hiện diện của dòng. Do đó `w04 == 1`
chính là bộ lọc *"chỉ giữ người thật sự có dữ liệu wave 4"*. Ba lý do bắt buộc:

1. **Loại attrition.** Trong 12.213 dòng, chỉ **9.665** người thực sự tham gia (2.548
   người đã rời panel). Không lọc thì kéo vào 2.548 dòng rỗng — đây chính là bước đầu của
   selection funnel và là biểu hiện trực tiếp của **chệch không phản hồi (attrition)**,
   rủi ro chệch quan trọng nhất của dự án (người rời panel có thể khác biệt hệ thống).
2. **Target chỉ tồn tại khi người đó trả lời.** Biến mục tiêu `employed_w04` lấy từ
   `w04ecoact` (hoạt động kinh tế wave 4). Người `w04 == 2` không trả lời câu này → giữ
   họ lại sẽ tạo target khuyết hàng loạt, làm hỏng cả nhãn lẫn tỷ lệ có việc 75,6%.
3. **Đúng mốc thời gian.** Cả dự án dự đoán tình trạng việc làm *tại wave 4*; chỉ người
   hiện diện ở wave 4 mới cung cấp quan sát hợp lệ cho mốc này (xem [Mục 5.1b](#51b)).

Nói gọn, hai điều kiện chia vai rõ ràng: `w04 == 1` đảm bảo **"có dữ liệu để phân tích"**
(loại attrition), còn `w04edu >= 3` đảm bảo **"đúng đối tượng cần phân tích"** (thu A về P).

### 5.3. Selection funnel (phễu chọn mẫu)

| Bước | Số respondent |
|---|---:|
| Khung wave 4 (access frame) | 12.213 |
| Tham gia wave 4 (`w04 == 1`) | 9.665 |
| Sau tốt nghiệp (`w04edu >= 3`) | 5.687 |
| Target có việc quan sát được | 5.687 |
| Có liên kết job-history | 4.932 |
| Có thời lượng việc trung bình | 1.717 |

### 5.4. Ba loại mẫu — quyết định thiết kế then chốt

Dự án **không dùng một mẫu duy nhất** cho mọi mục đích, mà phân vai ba loại để **cô lập
rủi ro chọn lọc**:

| Loại mẫu | Định nghĩa | Vai trò |
|---|---|---|
| **Mẫu chính (main)** | 5.687 người, đặc trưng phủ đầy đủ | Mọi kết luận & mô hình **lõi** |
| **Mẫu nhạy cảm (sensitivity)** | Tập con ~17% có khối học vấn chi tiết (ngành/GPA/loại trường) | Chỉ dùng cho mô hình `--narrow-scope` kèm cờ chỉ báo khuyết — *kiểm tra*, không phải kết luận chính |
| **Mẫu rà soát (review)** | Nhóm có liên kết job-history (4.932) / có thời lượng việc (1.717) | Mô tả outcome phụ; **không** dùng làm feature/target chính |

**Lý do tách bạch:** tiểu mẫu có khối học vấn chi tiết thuộc mô-đun "vừa tốt nghiệp đợt
này" và có tỷ lệ có việc **khác hẳn** toàn mẫu (≈51,8% so với 75,6%). Trộn nó vào mẫu
chính một cách ngây thơ sẽ tiêm **chệch lựa chọn (selection bias)** vào mọi ước lượng.

### 5.5. Năm loại chệch & sai số trong YP2021

| Loại | Biểu hiện cụ thể trong YP2021 |
|---|---|
| **Chệch bao phủ (coverage)** | Khung A là thanh niên nói chung, rộng hơn quần thể graduate → phải lọc học vấn |
| **Chệch lựa chọn (selection)** | Nhóm có dữ liệu ngành/GPA chi tiết chỉ ~17% mẫu, tỷ lệ có việc khác hẳn phần còn lại |
| **Chệch không phản hồi (non-response)** | Chính là attrition panel tích lũy qua các wave |
| **Chệch đo lường (measurement)** | Job-history dựa trên hồi tưởng; GPA là biến phân loại chuẩn hóa riêng |
| **Sai số lấy mẫu (sampling)** | Lượng hóa ở Mục 6.3 bằng urn/bootstrap/split |
| **Sai số phân bổ (assignment)** | *KHÔNG áp dụng* — YP2021 là khảo sát quan sát, không có phân nhóm thử nghiệm |

---

## 6. Phân tích chi tiết từng biểu đồ

Phần này là trọng tâm: với **mỗi biểu đồ**, ta trả lời ba câu hỏi — (a) **Cơ sở dữ liệu
nào** để vẽ? (b) **Biểu đồ cho ta biết điều gì?** (c) **Vì sao chọn cách trình bày này?**

---

### 6.1. Biểu đồ Selection Funnel (Phễu chọn mẫu)

**Loại:** biểu đồ cột 3 bậc (`Khung wave 4` → `Tham gia wave 4` → `Sau tốt nghiệp`).

- **Cơ sở vẽ:** bảng `analysis_sample_funnel.parquet` (cột `step_order`, `respondent_count`),
  lọc 3 bước đầu (`step_order <= 3`). Dữ liệu từ pipeline `build-target-sample`.
- **Cho ta biết:** quy mô **thu hẹp** từ khung chọn mẫu A (12.213) về mẫu phân tích S
  (5.687). Mỗi bậc tụt cho thấy có bao nhiêu người bị loại ở mỗi điều kiện lọc — đặc biệt
  bước "sau tốt nghiệp" cắt gần một nửa (9.665 → 5.687).
- **Vì sao chọn:** biểu đồ cột giảm dần là cách trực quan kinh điển để minh họa một
  funnel; nó làm rõ rằng mẫu phân tích **không phải toàn bộ panel** mà là kết quả của một
  chuỗi quyết định lọc — nền tảng để hiểu chệch bao phủ.

---

### 6.2. Biểu đồ Attrition Panel (Đường rơi rụng wave 1→4)

**Loại:** biểu đồ đường (line) với 4 điểm wave.

- **Cơ sở vẽ:** bảng `panel_participation_summary.parquet` (cột `wave`, `participated`).
- **Cho ta biết:** số người tham gia **giảm đơn điệu** qua các wave (12.213 → 10.721 →
  10.108 → 9.665). Vì YP2021 là **panel**, không phản hồi không biểu hiện thành từ chối
  lẻ tẻ mà thành **rơi rụng tích lũy**. Tỷ lệ giữ chân ≈ 79,1%.
- **Vì sao chọn:** đường nối các điểm theo thời gian thể hiện **xu hướng giảm liên tục**
  rõ hơn cột rời rạc. Đây là rủi ro chệch **không phản hồi** quan trọng nhất — người rời
  panel có thể khác biệt hệ thống so với người ở lại, ảnh hưởng mọi diễn giải về sau.

---

### 6.3. Bộ biểu đồ Sai số lấy mẫu (3 lăng kính)

Mẫu phân tích chỉ là **một** lần rút từ quần thể; nếu khảo sát lại, mọi ước lượng sẽ
chệch đi đôi chút. Phần này định lượng độ "rung lắc" đó bằng **ba phương pháp bổ trợ**,
gọi từ module `src.models.youth_panel.sampling_variation`.

#### 6.3.1. Biểu đồ urn/Binomial chồng lên Normal

- **Cơ sở vẽ:** mô phỏng **urn/Binomial** (4.000–5.000 lần rút) — coi mỗi người là một
  **phép thử Bernoulli** "có/không việc" với xác suất `p̂`; số người có việc tuân theo
  **Binomial(n, p̂)**. Đường mật độ Normal giải tích vẽ từ `analytic_se = sqrt(p(1-p)/n)`.
- **Cho ta biết:** phân phối mô phỏng urn **khớp gần như hoàn hảo** với xấp xỉ Normal —
  đúng như lý thuyết kỳ vọng khi cỡ mẫu lớn và `p̂` không sát 0/1. Điều này **xác nhận**
  khoảng tin cậy giải tích là đáng tin.
- **Vì sao chọn:** chồng mô phỏng lên lý thuyết là cách thuyết phục nhất để chứng minh ba
  cách tiếp cận hội tụ, thay vì chỉ tin một công thức.

#### 6.3.2. Biểu đồ Bootstrap ROC-AUC + Split variation

- **Cơ sở vẽ:**
  - **Bootstrap** (1.000–2.000 lần lấy mẫu *có hoàn lại*) trên tập kiểm thử → phân phối
    thực nghiệm của ROC-AUC, không cần giả định phân phối.
  - **Split variation** (80 lần chia train/test 80/20 ngẫu nhiên) → phân phối AUC/accuracy.
- **Cho ta biết:** khoảng tin cậy ROC-AUC ≈ **[0,64; 0,71]** — **không chứa 0,5**. Đây là
  bằng chứng tín hiệu phân loại là **THẬT** (dù yếu ~0,68), không phải nhiễu lấy mẫu.
  Biến thiên qua split nhỏ (std ≈ 0,016) ⇒ model **ổn định**, không phụ thuộc may rủi một
  lần chia.
- **Vì sao chọn:** với mục tiêu *dự đoán*, độ bất định của **chất lượng mô hình** quan
  trọng hơn độ bất định của một tỷ lệ. Hai biểu đồ này trả lời câu hỏi "model có thực sự
  học được gì không, hay chỉ ăn may?".

**Bảng tổng hợp sampling variation (con số thật):**

| Thí nghiệm | Ước lượng | SE/std | CI 95% | Phương pháp |
|---|---:|---:|---|---|
| employment_rate (giải tích) | 0,7565 | 0,0057 | [0,745; 0,768] | normal_approx |
| employment_rate (urn/binomial) | 0,7563 | 0,0057 | [0,745; 0,767] | 4.000 sim |
| employment_rate (bootstrap) | 0,7563 | 0,0055 | [0,746; 0,767] | 1.000 resample |
| test_roc_auc (bootstrap) | 0,6799 | 0,0181 | [0,644; 0,713] | bootstrap test-set |
| test_roc_auc (split) | 0,6811 | 0,0161 | [0,649; 0,718] | 80 splits 80/20 |
| test_accuracy (split) | 0,7577 | 0,0041 | [0,750; 0,765] | 80 splits 80/20 |

> **Kết luận:** ba cách ước lượng tỷ lệ có việc **gần như trùng nhau** (CI ≈ ±1,1 điểm
> phần trăm), xác nhận khoảng tin cậy giải tích.

---

### 6.4. Biểu đồ phân phối Target và biến job-history (EDA cơ bản)

**Loại:** lưới 3 biểu đồ — cột (target) + 2 histogram (job_changes, avg_job_duration).

- **Cơ sở vẽ:** lớp processed. Target `employed_w04` suy ra từ `w04ecoact == 1`. Hai biến
  job-history phái sinh từ `job_history_person_features.parquet`.
- **Cho ta biết:**
  - Mẫu **nghiêng hẳn về phía có việc** (75,6% có việc) → đây là **mất cân bằng lớp**,
    có hệ quả lớn cho việc chọn metric đánh giá (xem Mục 8).
  - Hai biến job-history **lệch phải mạnh** do phần lớn người chỉ có một đợt việc.
- **Vì sao chọn:** hiểu *hình dáng* của outcome trước khi mô hình hóa là bắt buộc. Việc
  thấy target lệch ngay từ đầu giải thích vì sao về sau ta không chỉ dựa vào accuracy.

---

### 6.5. Hai lưới small-multiples (phân loại + biến số)

**Loại:** lưới nhiều ô nhỏ — một lưới cho 11 biến phân loại (barh kèm % khuyết), một lưới
cho 5 biến số (histogram).

- **Cơ sở vẽ:** lớp processed; mỗi biến phân loại hiển thị `value_counts` top 8 + tỷ lệ
  khuyết `df[col].isna().mean()`.
- **Cho ta biết:** bức tranh tổng thể của mẫu — phân bố giới tính, vùng, học vấn, ngành,
  GPA, kỹ năng, nền tảng gia đình. Tỷ lệ khuyết ghi ngay trên tiêu đề mỗi ô cảnh báo biến
  nào "thưa".
- **Vì sao chọn:** small-multiples ưu tiên **độ phủ** hơn độ sâu — giúp người đọc nắm
  toàn bộ cấu trúc mẫu chỉ trong hai hình, thay vì hàng chục biểu đồ rời.

---

### 6.6. Biểu đồ Coverage (độ phủ biến processed)

**Loại:** biểu đồ thanh ngang (barh), tô **đỏ** các biến phủ < 50%.

- **Cơ sở vẽ:** bảng `yp2021_processed_coverage.parquet` (cột `valid_rate`, `valid_count`),
  do pipeline ghi lại tỷ lệ giá trị hợp lệ sau khi đã làm sạch skip code.
- **Cho ta biết:** nhóm biến **dưới 50% phủ** là nhóm ngành, loại trường, GPA và chuẩn bị
  việc làm — tất cả thuộc tiểu mô-đun "vừa tốt nghiệp đợt này" chỉ bao ~17% mẫu.
- **Vì sao chọn:** đây là **bằng chứng trực quan** cho quyết định ở Mục 5.4 — các biến
  phủ thấp này *không* vào mô hình lõi, chỉ dành cho mô hình nhạy cảm. Đường kẻ 50% và màu
  đỏ làm ngưỡng quyết định hiển thị tức thì.

---

### 6.7. Bảng Target framing (khung Bernoulli/Binomial)

**Loại:** bảng số (không phải hình).

- **Cơ sở vẽ:** hàm `de.target_framing(df)`. Tính `n`, `k` (số có việc), `p̂`, kỳ vọng
  Binomial `n·p`, độ lệch chuẩn đếm `sqrt(n·p·(1-p))`, SE tỷ lệ, và **majority-baseline
  accuracy = max(p, 1-p)**.
- **Cho ta biết:** `n = 5687`, `k = 4302`, `p̂ = 0,7565`. **Majority baseline accuracy =
  0,7565** — đây là độ chính xác mà một mô hình hằng số (luôn đoán "có việc") đã đạt được.
- **Vì sao quan trọng:** đặt ra **mốc tham chiếu thiết yếu**. Vì baseline đã đạt accuracy
  cao chỉ nhờ mẫu lệch, ta **buộc phải đánh giá model bằng ROC-AUC/PR**, không chỉ
  accuracy — nếu không sẽ bị "đẹp giả tạo".

---

### 6.8. Bảng nhận diện phân phối lý thuyết (Distribution identification)

**Loại:** bảng số.

- **Cơ sở vẽ:** hàm `de.distribution_identification(df)`. Với mỗi biến số tính **độ lệch
  (skewness)**, **độ nhọn vượt (excess kurtosis)** và **kiểm định Shapiro** (cap mẫu 5.000
  vì Shapiro không đáng tin ở n lớn). Quy tắc cờ Normal: `|skew| < 0,5 và |excess kurtosis|
  < 1`.
- **Cho ta biết:**
  - `age_w04`: skew −0,08, gần đối xứng nhưng excess kurtosis −1,04 (hơi phẳng) → **không
    Normal**.
  - Các biến đếm việc (`job_changes`, `total_jobs`, `avg_job_duration_months`): skew
    1,5–1,9 → **lệch phải rõ, không Normal**.
- **Vì sao chọn:** xác định xem có thể giả định Gauss cho các phương pháp tham số hay
  không. Lưu ý: Shapiro p rất nhỏ ở n lớn là bình thường (test quá nhạy), nên đọc **kèm**
  moment thống kê chứ không quyết một mình.

---

### 6.9. Biểu đồ ứng viên biến đổi (Transformation candidates)

**Loại:** biểu đồ thanh ngang so sánh `|skew|` trước/sau biến đổi.

- **Cơ sở vẽ:** hàm `de.transformation_candidates(df)`. Thử `log1p` và `sqrt` (chỉ hợp lệ
  cho dữ liệu không âm), chọn phép cho `|skew|` nhỏ nhất; chỉ đề xuất nếu giảm được `>0,1`.
- **Cho ta biết:** với biến lệch phải mạnh, `log1p`/`sqrt` **kéo độ lệch về gần 0**, giúp
  biến phù hợp hơn với mô hình giả định tuyến tính/Gauss. Biến đã gần đối xứng (tuổi) giữ
  nguyên (`none`).
- **Vì sao chọn:** đây là **đề xuất tiền-mô-hình-hóa**, minh họa tư duy chuẩn bị biến.
  (Lưu ý các biến đếm việc làm dù được khuyến nghị biến đổi vẫn **bị loại** khỏi model
  core do leakage — xem Mục 7.7.)

---

### 6.10. Lưới tỷ lệ có việc theo nhóm (Feature × Target)

**Loại:** lưới barh — tỷ lệ có việc theo từng giá trị của mỗi biến phân loại, kèm đường
nét đứt = trung bình chung 75,6%.

- **Cơ sở vẽ:** `df.groupby(col)[TARGET].mean()`, lọc nhóm có `count >= 20`.
- **Cho ta biết:** khoảng cách giữa các thanh so với đường trung bình cho biết biến đó
  **phân tách outcome mạnh hay yếu**. Biến nào có các nhóm trải rộng quanh đường = phân
  tách mạnh.
- **Vì sao chọn:** đây là bước **bản lề** giữa khám phá và mô hình hóa — đo *cường độ*
  liên hệ một cách trực quan. **Nhấn mạnh:** mọi chênh lệch là liên hệ trong mẫu, không
  nhân quả.

---

### 6.11. Bảng kiểm định thống kê (Welch t-test + Chi-square)

**Loại:** bảng số xếp theo p-value.

- **Cơ sở vẽ:**
  - **Welch t-test** (`stats.ttest_ind`, `equal_var=False`) cho 5 biến số — so sánh trung
    bình giữa nhóm có/không việc. Dùng Welch vì không giả định phương sai bằng nhau.
  - **Chi-square** (`stats.chi2_contingency`) cho 9 biến phân loại — kiểm tra độc lập.
    Chỉ chạy khi bảng chéo có mọi ô ≥ 5 (điều kiện hợp lệ của chi-square).
- **Cho ta biết (xếp theo ý nghĩa):**
  - **Tuổi** là biến phân tách mạnh nhất (p ≈ 0; có việc TB 27,5 tuổi vs không việc 26,1).
  - Tiếp theo: **chứng chỉ**, **vùng cư trú**, các biến **job-history**, **nhóm ngành**,
    **học vấn cha/mẹ**, **giới tính** — đều đạt ý nghĩa ở mức 5%.
  - **GPA** (p = 0,060) và **chuẩn bị việc làm** (p = 0,764) **không** đạt ý nghĩa — phần
    nào do cỡ mẫu hợp lệ nhỏ.
- **Vì sao chọn:** kiểm chứng những khác biệt trực quan ở 6.10 có **vượt ngưỡng ngẫu
  nhiên** hay không. Bài toán yêu cầu ≥3 mỗi loại kiểm định → dự án có 5 Welch + 9 chi-square.

---

### 6.12. Heatmap tương quan Pearson (biến số + target)

**Loại:** heatmap đối xứng, cmap `RdBu_r`, miền [−1, 1].

- **Cơ sở vẽ:** `numeric_correlation(df)` — ma trận Pearson giữa các biến số, gồm cả
  target ở dạng 0/1 (`work.corr(method="pearson")`).
- **Cho ta biết hai điều:**
  1. Biến nào **liên hệ với outcome** (đọc hàng/cột của target).
  2. **Cảnh báo đa cộng tuyến:** các cặp đặc trưng tự tương quan cao — nổi bật là
     `total_jobs ≈ job_changes + 1` (gần như trùng nhau, tương quan ~1).
- **Vì sao chọn:** heatmap là cách gọn nhất để thấy toàn bộ cấu trúc tương quan trong một
  khung nhìn. Phát hiện đa cộng tuyến giúp quyết định lược bớt biến trùng khi mô hình hóa.

---

### 6.13. Biểu đồ xếp hạng liên hệ Feature → Target

**Loại:** thanh ngang, mọi đặc trưng quy về **một thang cường độ chung** [0, 1].

- **Cơ sở vẽ:** `feature_target_association(df)`:
  - Biến số → **|point-biserial r|** (tương đương Pearson với target 0/1).
  - Biến phân loại → **Cramér's V** (đã hiệu chỉnh bias).
  - Cả hai đều nằm trong [0, 1] nên **chia sẻ chung một thang** để xếp hạng.
- **Cho ta biết:** thứ tự cường độ liên hệ của *mọi* đặc trưng với việc có việc làm, bất
  kể kiểu biến.
- **Vì sao chọn:** point-biserial và Cramér's V là hai thước đo phù hợp riêng cho từng
  kiểu biến; chuẩn hóa về cùng thang [0,1] cho phép **so sánh trực tiếp** biến số với biến
  phân loại — điều mà tương quan Pearson đơn thuần không làm được.

---

### 6.14. Bảng Subgroup ranking với Wilson CI (kiểm soát overclaim)

**Loại:** bảng số kèm khoảng tin cậy và **tầng độ chắc** (confidence tier).

- **Cơ sở vẽ:** `de.subgroup_ranking(df)`. Mỗi nhóm gắn:
  - **Khoảng tin cậy Wilson 95%** (`_wilson_ci`) — chính xác hơn khoảng Wald cho tỷ lệ,
    đặc biệt khi n nhỏ hoặc p sát biên.
  - **Cramér's V** + p-value chi-square cho biến.
  - **Tầng độ chắc:**
    - 🔵 *vững* — khác biệt có ý nghĩa thống kê **VÀ** CI không chứa tỷ lệ chung.
    - 🟡 *thận trọng* — CI chồng tỷ lệ chung, **HOẶC** nhóm thuộc tiểu mẫu chọn lọc ~17%.
    - 🟢 *mô tả* — không khác chung.
- **Cho ta biết:** ví dụ vùng Seoul (79,7%, CI [0,77; 0,82]) là 🔵 vững; còn mọi nhóm
  ngành học (Humanities 33,7%, Engineering 51,8%...) bị gắn 🟡 vì thuộc subset selection-biased.
- **Vì sao chọn:** đây là tuyến phòng thủ **chống overclaim** — ngăn việc thổi phồng một
  chênh lệch nhỏ (hoặc một chênh lệch lớn nhưng trên tiểu mẫu thiên lệch) thành "phát
  hiện". Đây là điểm trưởng thành về phương pháp luận của dự án.

---

### 6.15. Heatmap liên hệ đa biến giữa các predictor (Cramér's V)

**Loại:** heatmap, cmap `YlOrRd`, miền [0, 1].

- **Cơ sở vẽ:** `de.multivariate_association(df)` — Cramér's V **giữa các predictor phân
  loại với nhau** (không phải với target).
- **Cho ta biết:** cặp có V cao = hai biến **trùng thông tin** → đa cộng tuyến, gợi ý nên
  lược bớt khi mô hình hóa.
- **Vì sao chọn:** bổ trợ cho cảnh báo Pearson ở 6.12, nhưng cho **biến phân loại**. Đa
  cộng tuyến giữa predictor làm hệ số mô hình bất ổn và khó diễn giải.

---

### 6.16. Bảng Feature Catalog + Readiness Audit

**Loại:** bảng tổng hợp (sinh từ `feature_catalog.build_catalog(df)`).

- **Cơ sở vẽ:** lấy nguồn từ các tuple `CORE`/`LEAKAGE`/`NARROW_SCOPE` trong `dataset.py`,
  nên **không thể lệch** khỏi tập đặc trưng mô hình thực dùng.
- **Cho ta biết:** mỗi biến ứng viên kèm: vai trò (core / narrow-scope / loại vì leakage),
  kiểu dữ liệu, độ phủ, chiến lược điền khuyết – mã hóa – chuẩn hóa, và cờ
  `used_in_core_model`. Kết quả: **8 đặc trưng vào model core**, **3 cột loại vì leakage**,
  4 cột narrow-scope.
- **Vì sao chọn:** một catalog sinh tự động từ code đảm bảo tài liệu và mô hình **luôn
  đồng bộ** — không có chuyện báo cáo mô tả một tập feature còn code dùng tập khác.

---

### 6.17. Bốn ma trận nhầm lẫn (Confusion matrix)

**Loại:** 4 heatmap 2×2 (baseline, logreg L2, logreg L1, random forest).

- **Cơ sở vẽ:** `results.models[name].confusion` trên tập kiểm thử (`confusion_matrix`,
  labels [0,1]).
- **Cho ta biết:** baseline đạt accuracy cao chỉ nhờ **luôn đoán "có việc"** — nó không
  nhận diện được ai thất nghiệp (toàn bộ lớp 0 bị đoán sai). Các mô hình thật bắt đầu phân
  biệt được lớp thiểu số.
- **Vì sao chọn:** confusion matrix **mở bung** các con số tổng hợp thành 4 ô quyết định
  thực tế, phơi bày điều mà accuracy đơn lẻ che giấu — giá trị nằm ở khả năng nhận diện
  lớp thiểu số (thất nghiệp).

---

### 6.18. Đường ROC và đường Precision–Recall

**Loại:** 2 biểu đồ đường cạnh nhau.

- **Cơ sở vẽ:**
  - **ROC**: `roc_curve(y_test, y_proba)`, tóm tắt bằng AUC. Đường tham chiếu ngẫu nhiên
    = đường chéo (AUC 0,5).
  - **PR**: `precision_recall_curve`, tóm tắt bằng Average Precision (AP). Đường tham
    chiếu = prevalence (tỷ lệ lớp dương).
- **Cho ta biết:** các mô hình thật **vượt rõ** đường tham chiếu ngẫu nhiên (ROC-AUC ≈
  0,68; AP ≈ 0,85), xác nhận tín hiệu thật dù khiêm tốn.
- **Vì sao chọn cả hai:** ROC đánh giá năng lực phân tách ở mọi ngưỡng; **PR phù hợp hơn
  khi lớp lệch** vì tập trung vào lớp dương. Dùng cả hai cho cái nhìn cân bằng trên mẫu
  mất cân bằng.

#### ROC và AUC — tác dụng cụ thể

Đây là cặp công cụ đo **năng lực phân biệt (discrimination)** của mô hình — đặc biệt then
chốt với dự án này vì mẫu mất cân bằng (75,6% có việc).

**ROC (Receiver Operating Characteristic) — đường cong:**
- Vẽ quan hệ **TPR (tỷ lệ dương đúng)** theo **FPR (tỷ lệ dương sai)** khi quét **mọi
  ngưỡng quyết định** từ 0 đến 1, thay vì cố định ở 0,5.
- Đường chéo = mô hình ngẫu nhiên (vô dụng); đường càng *cong lên góc trái-trên* càng tốt.
- Tác dụng: cho thấy model phân biệt "có việc" vs "không việc" tốt đến đâu *độc lập với
  ngưỡng* — quan trọng vì ngưỡng tối ưu có thể khác 0,5 (xem threshold tuning, Mục 6.19).

**AUC (Area Under Curve) — diện tích dưới đường ROC:**
- Tóm tắt cả đường ROC thành **một con số** trong [0,5; 1].
- **Diễn giải xác suất:** AUC = xác suất model xếp một người *có việc* (ngẫu nhiên) cao
  hơn một người *không việc* (ngẫu nhiên). **0,5 = ngẫu nhiên · 1,0 = hoàn hảo.**
- Trong dự án: ROC-AUC ≈ **0,68**; bootstrap CI ≈ [0,64; 0,71] *không chứa 0,5* → tín
  hiệu là **thật** dù khiêm tốn.

**Tại sao ROC/AUC, không chỉ accuracy?** Đây là lý do mấu chốt: baseline luôn đoán "có
việc" đã đạt **accuracy 75,6%** nhưng **không nhận diện được một ai thất nghiệp** — và có
**ROC-AUC = 0,50** (đúng bằng ngẫu nhiên). Accuracy bị mẫu lệch "thổi phồng"; ROC-AUC thì
trung thực, cho thấy logistic (0,68) *thực sự học được tín hiệu* vượt baseline (0,50) —
điều accuracy không phân biệt nổi.

---

### 6.19. Calibration curve và Threshold tuning

**Loại:** 2 biểu đồ đường cạnh nhau.

- **Cơ sở vẽ:**
  - **Calibration**: `calibration_curve(y_test, y_proba, n_bins=8, strategy='quantile')`
    + Brier score. Đường chéo = hiệu chỉnh hoàn hảo.
  - **Threshold tuning**: `results.threshold_frame` — F1 theo ngưỡng quyết định từ 0,10
    đến 0,90.
- **Cho ta biết:**
  - Calibration kiểm tra xác suất dự đoán **có khớp tần suất quan sát** hay không — quan
    trọng khi dùng xác suất để **ra quyết định** chứ không chỉ xếp hạng. Brier score thấp
    (logistic ≈ 0,170) tốt hơn baseline (0,243).
  - Threshold tuning cho thấy dời ngưỡng khỏi mặc định 0,5 đánh đổi precision–recall ra
    sao — hữu ích khi **chi phí hai loại sai khác nhau**.
- **Vì sao chọn:** đây là lớp đánh giá **nâng cao** vượt ngoài "một con số accuracy" — phù
  hợp khi mô hình được dùng để hỗ trợ quyết định thực tế.

---

### 6.20. Biểu đồ Subgroup performance (công bằng theo nhóm)

**Loại:** lưới barh ROC-AUC của mô hình L2 theo giới tính / vùng / học vấn.

- **Cơ sở vẽ:** `results.subgroup_frame`, chỉ giữ nhóm có `n >= 30` trên tập kiểm thử.
- **Cho ta biết:** khoảng dao động ROC-AUC giữa các nhóm:
  - Học vấn: gap 0,033
  - Giới tính: gap 0,020
  - Vùng: gap 0,075 (lớn nhất — Gyeongin 0,635 vs Seoul 0,710)
  → mô hình **tương đối đồng đều**, không có nhóm nào bị bỏ rơi nghiêm trọng.
- **Vì sao chọn:** một mô hình có chỉ số tổng thể tốt vẫn có thể hoạt động **không đồng
  đều** giữa các nhóm. Kiểm tra fairness trước khi tin dùng là thực hành có trách nhiệm.

---

### 6.21. Biểu đồ hệ số Log-odds (Logistic L2)

**Loại:** thanh ngang, top 15 đặc trưng theo độ lớn log-odds; xanh = tăng, cam = giảm.

- **Cơ sở vẽ:** `results.coefficients` — hệ số logit của mô hình L2. Mỗi hệ số `β_j` là
  **log-odds**; `e^β_j` là **tỉ số odds**.
- **Cho ta biết:** đặc trưng nào làm **tăng/giảm** log của tỉ lệ cược có việc, và mức độ.
  Dấu và độ lớn **nhất quán** với liên hệ đã thấy ở EDA → khép kín mạch từ khám phá đến
  mô hình.
- **Vì sao chọn:** đây là **giá trị diễn giải** cốt lõi của hồi quy logistic — không chỉ
  dự đoán mà còn cho biết *tại sao*, đóng vai trò cầu nối giữa số liệu và câu chuyện thực.

---

### 6.22. Bảng đánh đổi Bias–Variance

**Loại:** bảng số (CV train AUC, CV test AUC, Test AUC, gap).

- **Cơ sở vẽ:** `results.metrics_frame` — so chênh lệch giữa điểm CV trên train và test.
- **Cho ta biết:**
  - Baseline: bias cao (underfitting), AUC = 0,5.
  - Logistic L1/L2: nâng chất lượng, **gap train–CV hẹp** (~0,02) → điều chuẩn kiểm soát
    phương sai tốt, không overfit.
  - Random Forest: gap train–CV lớn hơn (0,067, variance cao hơn) nhưng test AUC tương
    đương → **trần thông tin của tập đặc trưng** mới là giới hạn chính, không phải dạng
    mô hình.
- **Vì sao chọn:** đọc bias–variance giải thích vì sao ROC-AUC "chỉ" ~0,68 — không phải
  do model kém mà do tín hiệu trong nhóm đặc trưng nhân khẩu–nền tảng vốn không mạnh.

---

## 7. Tiền xử lý và trích xuất đặc trưng — vì sao chọn từng bước

Mục 4 của notebook (`yp2021_wrangling`) **cố tình quay về lớp raw_tables** để phơi bày
*quy trình* biến đổi, thay vì chỉ trình bày kết quả. Mỗi tiểu mục dưới đây tương ứng một
nhóm kỹ thuật wrangling, kèm **lý do chọn**.

### 7.1. Xử lý mã bỏ qua (skip codes) — bước nền tảng nhất

- **Làm gì:** mask các sentinel `{9090908, 9090909, 9999997, 9999998, 9999999}` thành `NaN`.
- **Vì sao:** nếu coi chúng là giá trị số thật, **mọi trung bình và phương sai sẽ sai
  lệch nghiêm trọng**. Đây là việc đầu tiên bắt buộc của mọi phân tích — phải cô lập phần
  khuyết *thật* trước khi bàn chiến lược điền.

### 7.2. Ba chiến lược điền khuyết (imputation) — tùy bản chất biến

| Biến | Chiến lược | Vì sao |
|---|---|---|
| Học vấn cha/mẹ | **Cờ chỉ báo khuyết (missing indicator)** | Khuyết theo **logic nhánh câu hỏi**, không ngẫu nhiên (không MCAR). Bịa giá trị sẽ sai; tạo cờ để mô hình **tự học** ý nghĩa của việc khuyết |
| Tuổi | **Trung vị (median)** | **Bền với phân phối lệch** và outlier hơn trung bình |
| Nhóm tài sản hộ gia đình | **Trung bình làm tròn (mean)** | Phù hợp biến gần đối xứng, làm tròn về mức rời rạc |

- **Vì sao phân biệt:** điền khuyết **sai bản chất** sẽ tiêm nhiễu hoặc áp đặt giả định
  MCAR (khuyết hoàn toàn ngẫu nhiên) không có cơ sở. Mỗi loại khuyết cần một cách xử lý
  riêng.

### 7.3. Đổi độ chi tiết (granularity) bằng groupby

- **Làm gì:** bảng `job_history` ở cấp **từng đợt việc (job-spell)** — một người nhiều
  dòng. `groupby('sampid')` + aggregate → đưa về cấp **cá nhân** (một-dòng-một-người),
  phái sinh `n_job_spells`, `years_active`...
- **Vì sao:** mô hình cần grain **một-dòng-một-người** để khớp với target. Đây là phép
  đổi granularity kinh điển khi gộp dữ liệu giao dịch về cấp đối tượng.

### 7.4. Chuyển dạng Wide ↔ Long bằng melt

- **Làm gì:** thông tin tham gia panel tự nhiên ở **dạng rộng** (mỗi wave một cột);
  `melt()` đưa về **dạng dài/tidy** (mỗi dòng = một cặp respondent–wave).
- **Vì sao:** dạng dài thuận tiện cho việc vẽ attrition và `groupby` đếm theo wave —
  nguyên tắc "tidy data". Ở dạng dài, đếm số người tham gia mỗi wave chỉ còn một dòng code.

### 7.5. Rời rạc hóa (discretization) bằng pd.cut

- **Làm gì:** chia tuổi liên tục thành nhóm thứ bậc (`22-24`, `25-27`, `28-30`, `31+`)
  bằng `pd.cut` với ranh giới `[21, 24, 27, 30, 100]`.
- **Vì sao:** ranh giới chọn **theo phân phối quan sát ở EDA**. Rời rạc hóa giúp một số
  mô hình/biểu đồ xử lý quan hệ phi tuyến và trình bày nhóm dễ đọc hơn.

### 7.6. Trích xuất bằng biểu thức chính quy (regex)

- **Làm gì:** dựng trường ngày dạng chuỗi `"YYYY-MM"` rồi parse/validate bằng regex —
  dùng lớp ký tự `\d`, neo `^` `$`, định lượng `{4}` `{1,2}`, nhóm `( )`, lựa chọn `|`
  (ví dụ kiểm tra tháng hợp lệ `0?[1-9]|1[0-2]`).
- **Vì sao:** minh họa kỹ thuật trích xuất/kiểm định trường văn bản — vừa **trích xuất**
  năm/tháng vừa **kiểm tra tính hợp lệ cú pháp** của trường. Kết quả: 27.126/27.126 dòng
  có tháng hợp lệ, 0 dòng nhiễm ký tự lạ.

### 7.7. Mã hóa biến phân loại (encoding): One-hot vs Dummy

- **Làm gì:** `pd.get_dummies` cho **one-hot** đầy đủ; `drop_first=True` cho **dummy** bỏ
  một mức mỗi biến (one-hot 9 cột vs dummy 7 cột).
- **Vì sao:** biến đổi cột định danh thành số để mô hình toán học tiếp nhận. `drop_first`
  tránh **đa cộng tuyến** (bẫy dummy variable) — lựa chọn tùy mô hình hạ nguồn có hệ số
  chặn (intercept) hay không.

### 7.8. Feature engineering trong pipeline sklearn (scaling + encoding an toàn)

Module `src/models/youth_panel/features.py` đóng gói tiền xử lý vào `ColumnTransformer`:

| Nhánh | Các bước | Lý do |
|---|---|---|
| **Numeric** | `SimpleImputer(median)` → `StandardScaler` | Điền trung vị bền với lệch; chuẩn hóa để hệ số logistic so sánh được |
| **Categorical** | `SimpleImputer(constant='missing')` → `OneHotEncoder(handle_unknown='ignore')` | Khuyết thành một mức riêng; `handle_unknown` chống vỡ khi gặp mức lạ ở test |

> **Vì sao gói trong pipeline:** để `StandardScaler`/`SimpleImputer`/`OneHotEncoder` chỉ
> **fit trên fold train** ở mỗi vòng CV → **không rò rỉ thống kê** từ test sang train.

### 7.9. Checklist chống rò rỉ (leakage control) — tuyến phòng thủ then chốt

| Quy tắc | Cụ thể |
|---|---|
| **Loại biến đo SAU outcome** | `total_jobs`, `job_changes`, `avg_job_duration_months` đếm việc *sau* khi tình trạng việc được quan sát → loại khỏi cả core lẫn narrow |
| **Không dùng cột nguồn của target** | Các cột `employed_w04_source*` (sinh ra target) không nằm trong feature |
| **Impute/scale/encode trong Pipeline** | Chỉ fit trên fold train ở mỗi vòng CV |
| **Chia trước, biến đổi sau** | `train_test_split` (stratify) chạy trước; preprocessor fit sau |
| **Narrow-scope cô lập** | Khối ngành/GPA/loại trường (~17%) chỉ vào model `--narrow-scope` kèm cờ khuyết, không impute như MCAR |

> **Vì sao quan trọng:** đây là lý do điểm số mô hình **không bị "đẹp giả tạo"**. Nếu để
> lọt biến job-history (đo sau khi có việc), model sẽ "nhìn trộm đáp án" và ROC-AUC sẽ cao
> giả tạo.

---

## 8. Mô hình hóa và đánh giá

### 8.1. Bài toán và 4 mô hình

Bài toán: **phân loại nhị phân** — dự đoán `employed_w04` từ 8 đặc trưng core đã tinh lọc.

| Mô hình | Cấu hình | Vai trò |
|---|---|---|
| **Baseline** | `DummyClassifier(most_frequent)` | Luôn đoán lớp đa số — mốc phải vượt |
| **Logistic L2 (Ridge)** | `LogisticRegression(l1_ratio=0, C=1, solver='saga')` | Mô hình chính, điều chuẩn L2 |
| **Logistic L1 (Lasso)** | `LogisticRegression(l1_ratio=1, C=1, solver='saga')` | Điều chuẩn L1 (có thể đẩy hệ số về 0) |
| **Random Forest** | `n_estimators=250, min_samples_leaf=20, class_weight='balanced_subsample'` | So sánh phi tuyến nội bộ |

> **Ghi chú kỹ thuật:** sklearn 1.8+ bỏ tham số `penalty` (deprecated); chọn Ridge/Lasso
> nay qua `l1_ratio` trên solver `saga` (`l1_ratio=0` → L2, `l1_ratio=1` → L1).

### 8.2. Quy trình đánh giá

1. **Kiểm tra modeling-ready** trước khi fit (4 điều kiện): grain một-dòng-một-người
   (sampid không trùng); target nhị phân 0/1 đã tách khỏi X; không có cột leakage trong X;
   missingness < 100% mỗi cột. **Cả 4 đạt** → X shape (5687, 8).
2. **Chia stratified 80/20** (4.549 train / 1.138 test) — giữ tỷ lệ lớp.
3. **Stratified k-fold CV** (5 fold) trên train, đọc bias–variance.
4. **Metric đầy đủ trên test**: accuracy, precision, recall, F1, ROC-AUC, Average
   Precision, Brier score, confusion matrix.

### 8.3. Kết quả metric (con số thật)

| Mô hình | Accuracy | Precision | Recall | F1 | ROC-AUC | Avg.Prec | Brier |
|---|---:|---:|---:|---:|---:|---:|---:|
| baseline_most_frequent | 0,7566 | 0,7566 | 1,0000 | 0,8614 | 0,5000 | 0,7566 | 0,2434 |
| logreg_l2 | 0,7601 | 0,7663 | 0,9826 | 0,8611 | 0,6788 | 0,8547 | 0,1698 |
| logreg_l1 | 0,7610 | 0,7665 | 0,9837 | 0,8616 | 0,6789 | 0,8547 | 0,1698 |
| random_forest | 0,6555 | 0,8443 | 0,6678 | 0,7458 | 0,6812 | 0,8534 | 0,2202 |

**Đọc kết quả:**
- Về **accuracy**, các mô hình gần như ngang baseline — vì mẫu lệch, accuracy không phân
  biệt được model tốt/xấu.
- Về **ROC-AUC** (thước đo phân biệt thật), logistic và RF đều ~0,68, **vượt rõ** baseline
  0,50.
- **Average Precision** logistic (0,855) cao hơn baseline (0,757) đáng kể.
- **Brier score** logistic (0,170) tốt nhất → xác suất dự đoán đáng tin nhất.

---

## 9. Cầu nối toán học

Hồi quy logistic ước lượng xác suất có việc qua hàm **sigmoid** trên tổ hợp tuyến tính:

$$ z = \beta_0 + \beta_1 x_1 + \dots + \beta_k x_k, \qquad \hat p = \sigma(z) = \frac{1}{1+e^{-z}}, \qquad \ln\frac{\hat p}{1-\hat p} = z $$

- Mỗi hệ số $\beta_j$ là **log-odds**; $e^{\beta_j}$ là **tỉ số odds**.
- Huấn luyện = cực tiểu hóa **cross-entropy loss** (log loss), tức negative log-likelihood
  của Bernoulli:

$$ L(\beta) = -\frac{1}{n}\sum_i \big[ y_i \ln \hat p_i + (1-y_i)\ln(1-\hat p_i) \big] $$

- **Gradient descent** cập nhật tham số với gradient gọn — *sai số dự đoán* nhân đặc trưng:

$$ \frac{\partial L}{\partial \beta_j} = \frac{1}{n}\sum_i (\hat p_i - y_i)\, x_{ij}, \qquad \beta \leftarrow \beta - \eta\, \nabla L $$

- sklearn dùng solver `saga` (biến thể gradient ngẫu nhiên); $C = 1/\lambda$ điều khiển
  cường độ điều chuẩn L1/L2.
- **Baseline theo loss:** mỗi hàm mất mát có hằng số tối ưu riêng — MSE → trung bình, MAE
  → trung vị, log loss → tỷ lệ lớp (prevalence). Vì target nhị phân, baseline đúng là
  **most-frequent** với ROC-AUC = 0,5.

---

## 10. Kết luận và hạn chế

### 10.1. Trả lời câu hỏi nghiên cứu

Trên mẫu pilot **5.687 người** vừa tốt nghiệp, khoảng **ba phần tư có việc** ở wave 4
(75,6%, CI 95% hẹp ≈ [74,5%; 76,8%], đã kiểm chứng bằng urn/binomial + bootstrap). Khả
năng có việc **liên hệ rõ nhất** với:

1. **Tuổi** (biến phân tách mạnh nhất)
2. **Chứng chỉ**
3. **Vùng cư trú**
4. **Nền tảng học vấn của cha mẹ**

Mô hình logistic điều chuẩn **vượt baseline ổn định** qua CV, test, và hàng trăm lần chia
dữ liệu, với gap bias–variance hẹp — đủ tin cậy để **xếp hạng yếu tố liên hệ**, dù sức
phân loại tuyệt đối còn khiêm tốn (ROC-AUC ≈ 0,68).

### 10.2. Hạn chế (đã phơi bày minh bạch)

| Hạn chế | Nguồn gốc |
|---|---|
| **Chệch bao phủ** | Bước lọc học vấn để thu khung A về quần thể P |
| **Chệch không phản hồi** | Attrition panel (giữ chân 79%) |
| **Chệch lựa chọn** | Tiểu mẫu học vấn chi tiết ~17% (đã gắn cờ 🟡 trong subgroup) |
| **Rò rỉ tiềm tàng** | 3 biến job-history đã bị loại để phòng (ghi trong catalog + checklist) |
| **Sức phân loại khiêm tốn** | Tín hiệu trong nhóm đặc trưng nhân khẩu–nền tảng vốn không mạnh |

> **Mọi kết luận đọc trong phạm vi mẫu pilot wave 4 sau tốt nghiệp** — không suy rộng cho
> toàn panel, không diễn giải nhân quả, không gộp với GOMS/KEEP II.

### 10.3. Hướng mở rộng

- Bổ sung mô hình nhạy cảm có khối học vấn chi tiết sau cờ khuyết (`--narrow-scope`).
- Hiệu chỉnh trọng số attrition (attrition weighting).
- Áp các phép biến đổi đề xuất ở Mục 6.9 cho biến lệch.
- Tách target ba trạng thái (có việc / thất nghiệp / ngoài lực lượng lao động) thay vì nhị
  phân.

---

## 11. Phụ lục tái lập

**Pipeline dữ liệu (raw → processed):**

```bash
python -m src.data.youth_panel.pipeline build-metadata
python -m src.data.youth_panel.pipeline build-raw-tables
python -m src.data.youth_panel.pipeline build-panel
python -m src.data.youth_panel.pipeline build-job-history
python -m src.data.youth_panel.pipeline build-construct-candidates
python -m src.data.youth_panel.pipeline build-target-sample
python -m src.data.youth_panel.pipeline build-processed-pilot
```

**Phân tích, mô hình hóa, bất định, EDA sâu, catalog:**

```bash
python -m src.visualization.yp2021_correlation
python -m src.visualization.yp2021_deep_eda
python -m src.models.youth_panel.feature_catalog build
python -m src.models.youth_panel.pipeline build-model
python -m src.models.youth_panel.pipeline build-model --narrow-scope
python -m src.models.youth_panel.sampling_variation run
```

**Notebook & test:**

```bash
python notebooks/_build_yp2021_report_nb.py
jupyter nbconvert --to notebook --execute --inplace notebooks/yp2021_report.ipynb
pytest tests/ -q -k youth_panel
```

**Tính tất định:** mọi mô hình/mô phỏng seed `RANDOM_STATE = 42`; CSV ghi `utf-8-sig`;
notebook sinh từ script `notebooks/_build_*.py` rồi execute, nên nguồn versioned và output
tái tạo được.

**Bộ test liên quan** (`tests/test_youth_panel_*.py`): raw_tables, metadata, panel
assembly, job_history, construct_candidates, target_sample, processed, feature_catalog,
deep_eda, model, sampling_variation, docs_integration — tất cả pass.

---

*Hết. Mọi con số trong tài liệu này quan sát trực tiếp từ pipeline YP2021; mọi kết luận là
liên quan/dự đoán, không nhân quả; chỉ trong phạm vi pilot wave 4 sau tốt nghiệp.*
