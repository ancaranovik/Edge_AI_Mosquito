# B — Baseline trên A_v1

Ngày ghi: 20/10/2026. **Đã duyệt: code + cài phụ thuộc + kiểm tra nhỏ. Đã triển khai notebook B và chạy smoke; chưa chạy baseline đầy đủ.**

File này giữ các quyết định và đề xuất cho phần B để tiếp tục công việc không mất context. Phạm vi B ở đây thay đề xuất ban đầu chỉ có logistic regression và CNN nhỏ; không thay đổi dữ liệu hay protocol A_v1 đã khóa.

## 1. B cần trả lời điều gì?

So sánh những cách học đủ khác nhau để tìm ứng viên cho ESP32-S3 + INMP441: đặc trưng phổ truyền thống, CNN trên log-mel, convolution theo thời gian, CNN học chuỗi và CNN trên waveform. B tạo mốc kết quả để C–E kiểm tra các cải tiến.

Mục tiêu không phải chạy thật nhiều tên model hoặc mặc định model từng thắng sẽ tiếp tục thắng. Model được chọn theo cả chất lượng từng loài và chi phí tính toán. Chưa xác nhận khả năng triển khai chỉ từ số tham số.

## 2. Kế thừa project trước

- Đã rà lại kết quả thực của bốn kiến trúc trong Colab baseline, notebook/snapshot và thư mục review local của project cũ. Nhận định trước đây rằng local chưa đủ kết quả đã được cập nhật: có đầy đủ artifact exp007/008, output exp002–006/010 và CSV dự đoán tải về. DS-CNN/03A exp007 là ứng viên tốt nhất về window/source F1 trong nhóm trial 004–008, nhưng thứ hạng thay đổi theo recipe và đơn vị đánh giá. Xem mục 12 và các CSV bằng chứng.
- Protocol cũ dùng arabiensis, funestus, squamosus, Culex pipiens complex. Frontend log-mel tham chiếu là 16 kHz, ngữ cảnh 0,975 giây. A_v1 hiện dùng bốn loài khác, 8 kHz, 300 ms và cách chia domain mới.
- Tái sử dụng ý tưởng/định nghĩa kiến trúc sau khi kiểm tra, không khởi tạo lại toàn bộ project cũ. Điều chỉnh kích thước đầu vào/pooling nếu cần và ghi rõ trong report.
- Huấn luyện lại từ đầu trên A_v1. Không nhập checkpoint, cache, nhãn hoặc điểm cũ làm kết quả mới. Transfer learning từ checkpoint cũ, nếu thử, là thí nghiệm riêng về sau.

## 3. Dữ liệu và ranh giới đánh giá giữ cố định

Đọc `data/manifests/biodcase_four_species_clips.csv`, `biodcase_four_species_windows.csv` và `results/A/preparation/tables/dataset_freeze.csv`. Kiểm tra version, checksum và nhãn trước khi lấy dữ liệu; dùng cột `split`.

| Loài | Train window | Validation window | Test window — chỉ ghi nhận từ A |
| --- | ---: | ---: | ---: |
| Aedes aegypti | 15.592 | 4.681 | 39.113 |
| Aedes albopictus | 4.122 | 164 | 7.886 |
| Anopheles dirus | 3.669 | 1.184 | 11.501 |
| Anopheles minimus | 18.454 | 5.923 | 1.188 |
| Tổng | 41.837 | 11.952 | 59.688 |

- Window 300 ms = 2.400 sample ở 8 kHz; hop giữa các window 150 ms. Dùng đúng quy trình mono/resample và biên độ float32 của A.
- B–E chỉ phát triển bằng train/validation. B được đọc metadata/checksum để xác minh; không tải audio test vào cache/loader, không dự đoán hoặc chấm điểm test. F mới đánh giá test sau khi cố định phương pháp và tham số.
- Không tạo split mới, không cắt giảm lớp lớn, không bổ sung D5 ngoài pool đã chọn. Mọi thống kê chuẩn hóa/imputation học từ dữ liệu chỉ fit trên train, không dùng nhãn để chuẩn hóa.
- Label ID giữ theo manifest: 0 aegypti, 1 albopictus, 2 dirus, 3 minimus. Không đưa tên file, đường dẫn, domain hoặc metadata nguồn vào đầu vào model B.

## 4. Danh sách baseline đề xuất: tám model

| Model | Đầu vào | Vai trò và phạm vi |
| --- | --- | --- |
| Small CNN | Log-mel | Mốc CNN cơ bản, kế thừa kiến trúc cũ |
| DS-CNN | Log-mel | Ứng viên ưu tiên từ kết quả cũ; đối chứng convolution tách depthwise/pointwise |
| TC-ResNet8 | Log-mel, giữ đúng trục thời gian | Đối chứng temporal convolution; ghi rõ khi thích nghi đầu vào |
| CNN-LSTM | Chuỗi đặc trưng sau CNN | Đối chứng học chuỗi; kiểm tra số bước thời gian còn lại và chi phí recurrent |
| Đặc trưng phổ + Random Forest | Log-PSD, thống kê phổ, F0/harmonic khi có ước lượng đáng tin | Mốc truyền thống; giới hạn độ sâu/số cây, ghi nhận F0 thiếu hoặc lỗi harmonic |
| CNN 1D theo MosquitoSong+ | Waveform 2.400 sample | Đối chứng waveform sát nghiên cứu muỗi; nếu thu nhỏ/chỉnh kiến trúc phải gọi rõ là biến thể |
| BC-ResNet-1 | Log-mel | Kiến trúc âm thanh nhỏ bổ sung; kiểm tra normalization, kích thước tần số và khả năng xuất model |
| Enhanced MicroDSC-style | Log-mel 40×27 | Bổ sung đã duyệt: đối chứng DS-CNN bằng kênh 16/32/48/64, ReLU6, pooling và GAP; ghi rõ thích nghi từ code v1.0.0, chưa tái lập paper |

Thêm mốc luôn đoán lớp có nhiều window train nhất, tie-break theo label ID. Đây là kiểm tra imbalance, không tính là model nghiên cứu thứ chín.

Chưa đưa logistic regression, SVM, thêm biến thể GRU hoặc nhiều họ ResNet khác vào danh sách bắt buộc. Chỉ bổ sung khi có câu hỏi thí nghiệm cụ thể và được duyệt.

### Bằng chứng và giới hạn của lựa chọn

- [DS-CNN — Hello Edge](https://arxiv.org/pdf/1711.07128): có bằng chứng hiệu quả trên nhận dạng từ khóa cho vi điều khiển. Kiến trúc thu nhỏ trong project không mặc định có cùng chất lượng/bộ nhớ với bản paper.
- [TC-ResNet](https://www.isca-archive.org/interspeech_2019/choi19_interspeech.pdf): temporal convolution nhỏ, được đánh giá trên Speech Commands và điện thoại; không phải điểm phân loại muỗi hoặc độ trễ ESP32-S3.
- [MosquitoSong+](https://journals.plos.org/plosone/article?id=10.1371/journal.pone.0310121): bằng chứng trực tiếp về tiếng muỗi, dùng 300 ms ở 8 kHz. Recipe trong paper còn có augmentation và xử lý dữ liệu riêng. B dùng recipe chung là so sánh kiến trúc thích nghi, chưa phải tái lập đầy đủ paper.
- [BC-ResNet](https://www.isca-archive.org/interspeech_2021/kim21l_interspeech.pdf): bản BC-ResNet-1 khoảng 9.200 tham số; kết quả công bố trên Speech Commands. Kích thước và chi phí của bản bốn lớp/300 ms phải đo lại, chưa chứng minh dùng được trên board.
- [Nghiên cứu đặc trưng muỗi](https://doi.org/10.1186/s13071-022-05324-5): cơ sở cho đối chứng PSD/F0/MFCC và ML truyền thống; cảm biến quang học và nhãn genus/giới tính khác bài hiện tại. RF không tự động nhẹ nếu nhiều cây/sâu.
- [MTRCNN BioDCASE](https://biodcase.github.io/challenge2026/task5): khoảng 0,22 triệu tham số, mô tả cho clip dài hơn 1,1 giây; chưa đưa vào B bắt buộc vì không dùng nguyên bản cho window 300 ms. Nếu bổ sung cần đánh dấu là bản thích nghi và duyệt riêng.

## 5. Các thông số triển khai đã duyệt

- Khai báo trực tiếp ở cell đầu; không thêm file config JSON. Seed ban đầu 42. Một lượt đầy đủ cho mỗi baseline, không dò grid ở B; đây chưa phải bằng chứng ổn định nhiều seed.
- Log-mel khởi đầu: FFT/window STFT 256 sample (32 ms), hop STFT 80 sample (10 ms), 40 mel bins, dải 0–4 kHz, không pad thời gian. Mỗi window 2.400 sample tạo 27 frame, tensor chuẩn ghi rõ trục frequency/time. Hop STFT 10 ms khác hop cắt dữ liệu 150 ms.
- Tính frontend bằng NumPy/SciPy để giảm phụ thuộc. Kiểm tra filter bank không rỗng, tín hiệu hữu hạn và đủ frame; không tái dùng ma trận mel 16 kHz của project cũ.
- Nhóm log-mel dùng cùng frontend và cùng quy tắc chuẩn hóa theo bin fit trên train; waveform dùng thống kê toàn cục từ train. RF chỉ impute giá trị thiếu bằng train; không mặc định đỉnh FFT là F0. Ghi định nghĩa đặc trưng, độ tin cậy F0 và trường hợp không ước lượng được.
- Neural model: đề xuất Adam, learning rate 0,001, batch 64, tối đa 40 epoch, early stopping patience 8 theo validation clip macro-F1; hòa điểm giữ epoch sớm hơn. Không label smoothing, không augmentation, natural sampling và cross-entropy không weight. Đây là recipe khởi đầu, không bảo đảm tối ưu công bằng cho mọi kiến trúc.
- RF: đề xuất 100 cây, max depth 10, seed 42, class_weight=None. Giữ nguyên toàn bộ pool training; ngân sách RF ghi riêng, không quy đổi epoch RF thành epoch neural model.
- Ghi số update, epoch, thời gian và điều kiện dừng thực tế. Cùng trần ngân sách/đầu vào không đồng nghĩa các model đã được tune tốt như nhau; tối ưu riêng và xác nhận nằm ở E–F.
- Chạy local trước. Hướng khởi đầu là PyTorch CPU; kiểm tra môi trường và dung lượng trước khi cài, không cài CUDA hoặc tự chuyển Colab. Nếu CPU quá chậm, báo thời gian smoke test để thống nhất nơi chạy, không tự đổi dữ liệu/ngân sách.

## 6. Notebook dự kiến làm những phần gì?

Tạo một notebook `notebooks/04_B_baseline.ipynb`, có tiêu đề và markdown giải thích từng khối:

1. Câu hỏi B, phạm vi và giới hạn; thiết lập trực tiếp.
2. Xác minh A_v1/checksum/label/split; công bố coverage và imbalance.
3. Đọc đúng window train/validation, kiểm tra shape và biên độ; khóa loader test khỏi quá trình B.
4. Tạo và minh họa log-mel, đặc trưng phổ, waveform; fit thống kê chỉ trên train.
5. Cache đặc trưng theo hash manifest + cấu hình frontend + phiên bản thực thi; không tải test. Chỉ đọc clip thật cần dùng, có cache audio giới hạn bộ nhớ; không ghi WAV nhỏ.
6. Khai báo tám model và bảng kiểm tra đầu vào, output bốn lớp, số tham số, trọng số. Nêu khác biệt so với code/paper gốc; riêng MicroDSC có bảng kích thước sau từng stage và đối chiếu cách đếm tham số.
7. Mốc majority và RF; fit đúng train, dự đoán validation.
8. Huấn luyện từng neural model, lưu/resume checkpoint có kiểm tra run identity, lịch sử và model được chọn. Không tái dùng cache/checkpoint khi dữ liệu/recipe khác.
9. Tổng hợp xác suất window theo clip; tính metric và so sánh.
10. Biểu đồ học, confusion matrix chuẩn hóa theo hàng kèm support, recall/F1 từng loài, bảng hiệu quả/chi phí.
11. Report kết quả, thất bại và giới hạn; gợi ý ứng viên cho C mà chưa tuyên bố model thắng ổn định.

Notebook có chế độ smoke test rõ ràng; output smoke tách khỏi kết quả đầy đủ và không được đưa vào bảng xếp hạng/report kết luận. Nếu chưa chạy full, report phải ghi chưa có kết quả full, không tạo số giả.

## 7. Đánh giá và cách đọc kết quả

- Chọn checkpoint neural bằng validation **clip-level macro-F1**: trung bình xác suất trên các window cùng clip, rồi argmax. Bốn nhãn cố định; clip ngắn hơn 300 ms không tính metric, nhưng ghi coverage.
- Báo thêm balanced accuracy, precision/recall/F1/support từng loài, confusion matrix và gap train/validation. Window-level metric là bổ sung, không coi overlap là các bản ghi độc lập.
- Theo domain: ghi support và accuracy/recall cho từng cặp loài–domain có dữ liệu. Không diễn giải macro-F1 của domain chỉ có một loài như độ chính xác phân biệt đủ bốn loài.
- Chi phí: số tham số, kích thước trọng số thô và checkpoint thực, MAC ước tính với phạm vi operator đã đếm, latency frontend + model trên CPU với số thread/batch/warmup được ghi rõ. RF báo số cây/node, kích thước lưu và latency.
- RAM activation nếu ước lượng chỉ là con số có phạm vi, không phải peak RAM ứng dụng. Latency PC không thay latency ESP32-S3; model FP32 nhỏ chưa chứng minh export int8/TFLite Micro thành công. Board có PSRAM hay không hiện chưa xác nhận.
- Không so trực tiếp điểm clip macro-F1 A_v1 với source macro-F1 project cũ hoặc accuracy từ paper/split khác.

## 8. Những hạn chế A phải xuất hiện trong report B

- Albopictus train D5, validation chỉ một nguồn D1 (67 clip đủ 300 ms, 164 window), test D2. Nhiều clip hoặc seed không tạo thêm nguồn validation độc lập. Thứ hạng theo validation này cần đọc thận trọng.
- Dirus train/validation chỉ D1, có ít window train nhất; minimus có khoảng 80% window train từ D4.
- A_v1 giữ domain theo từng loài; không phải domain hoàn toàn chưa có trong toàn bộ training. Chỉ tách được các nhóm nguồn/nội dung đã biết hoặc phát hiện; không chứng nhận độc lập cá thể.
- Cảm biến D5 và nhiều clip khác chưa có ánh xạ xác minh đầy đủ. Chưa gọi đây là benchmark microphone-only hoặc bằng chứng chuyển sang INMP441.
- Nhãn window kế thừa clip; A chưa xác nhận mọi window có wingbeat rõ. B chưa có nhãn giới tính, detection muỗi/nền hay class uncertain đã hiệu chuẩn; không tự thêm các nhãn đó.

## 9. File dự kiến tạo/sửa sau khi duyệt

| File/thư mục | Thay đổi |
| --- | --- |
| `notebooks/04_B_baseline.ipynb` | Tạo notebook tự đủ các phần nêu trên; không thêm một framework nhiều file |
| `requirements.txt` | Bổ sung/pin scikit-learn và torch CPU tương thích Python 3.12 cùng môi trường đang có; kiểm tra phụ thuộc. Không thêm librosa/torchaudio/torchvision nếu không cần |
| `.venv/` | Đã cài torch 2.14.1+cpu từ index CPU chính thức, scikit-learn 1.9.1, joblib 1.6.0, threadpoolctl 3.7.0; pip check không có xung đột |
| `results/B/cache/` | Cache train/validation; log-mel float32 40×27 khoảng 232 MB trước overhead, chưa gồm đặc trưng ML. Waveform không nhân bản thành WAV |
| `results/B/<smoke hoặc full>/models/<model>/` | Checkpoint/bộ tiền xử lý cần thiết, trọng số được chọn; hai chế độ tách riêng |
| `results/B/<smoke hoặc full>/tables/` | Cấu hình lần chạy dưới dạng bảng, metric, lịch sử, dự đoán validation và thống kê chi phí có model ID/seed |
| `results/B/<smoke hoặc full>/figures/` | Biểu đồ phục vụ report |
| `results/B/<smoke hoặc full>/report.html` | Report tự chứa hình và số liệu thực của chế độ chạy; không xếp hạng từ smoke |
| `results/B/report.html` | Trang chỉ dẫn đến report smoke/full đã có; chưa có full thì ghi rõ |
| `README.md` | Cập nhật cách chạy B và nơi xem kết quả sau khi triển khai |
| `B_PLAN.md` | Cập nhật trạng thái duyệt/triển khai; giữ một nơi ghi quyết định |

Không sửa notebook 01–03, manifests/freeze A, config.json, WAV gốc hay project cũ trong phạm vi B. Nếu gặp vấn đề buộc thay dữ liệu/protocol A, trình bày riêng trước khi sửa.

## 10. Phần còn lại của A–F

| Phần | Nội dung |
| --- | --- |
| A | Đã chốt A_v1 trong phạm vi thăm dò đã công bố |
| B | Tám baseline; lựa chọn ban đầu theo chất lượng và chi phí |
| C | Thay từng yếu tố: frontend, window/overlap, imbalance, noise/gain augmentation, thành phần nhỏ như attention nếu phù hợp |
| D | Kết hợp cải tiến; cân nhắc SupCon, alignment, pretrained embeddings/teacher và ensemble theo dữ liệu thực có |
| E | Dò tham số, thu nhỏ, distillation và lượng tử hóa các ứng viên có triển vọng |
| F | Xác nhận nhiều seed; đánh giá test sau khi cố định phương pháp; chưa thay cho thử nghiệm thiết bị |

Các phương pháp nâng cao chưa phải bắt buộc ghép hết. Alignment cần xét thiếu coverage loài–domain; augmentation không tạo thêm cá thể/nguồn thu độc lập. Phần sau điều chỉnh dựa trên kết quả phần trước.

## 11. Phê duyệt và bàn giao hiện tại

Người dùng đã chọn **phương án 1: code + kiểm tra nhỏ**. Đã tạo `notebooks/04_B_baseline.ipynb` với 17 mục, 17 code cell; mặc định `RUN_MODE='smoke'`. Đã cài thư viện CPU và kiểm tra 32 window train / 16 window validation từ 24 clip. Bảy neural model chạy hai epoch giới hạn hai batch/epoch; RF fit/predict trên phần train/validation nhỏ. Chưa tự khởi chạy full hoặc đánh giá test.

Các phạm vi đã trình:

1. **Code + cài phụ thuộc + kiểm tra nhỏ:** viết notebook đầy đủ, kiểm tra frontend/loader và forward/backward của mọi neural model, fit/predict RF bằng mẫu train/validation nhỏ. Chưa tự khởi chạy tám lượt training đầy đủ; notebook sẵn để Run All.
2. **Code + cài phụ thuộc + chạy baseline đầy đủ:** làm phần 1, sau đó huấn luyện/đánh giá validation cả tám model theo ngân sách trên và tạo report kết quả. Thời gian CPU chưa thể cam kết trước smoke test.
3. **Điều chỉnh kế hoạch:** sửa danh sách, thông số, cách chạy hoặc cấu trúc file trước khi triển khai.

Không tự coi thời gian chờ hoặc lựa chọn mặc định trên giao diện là phê duyệt.

Report smoke: `results/B/smoke/report.html`. Các metric smoke chỉ chẩn đoán, không dùng kết luận model tốt nhất. Các kiểm tra gồm tone/silence/gain, hash A, shape, gradient, serialization, resume lượt hoàn tất và chặn audio test. Chưa kiểm chứng mọi tình huống ngắt giữa epoch hoặc export lên ESP32-S3.

## 12. Bài học đã đối chiếu lại từ project cũ — 20/10/2026

### Phạm vi đọc và mức bằng chứng

Đã đọc các notebook 00–04, pipeline/frontend contracts, cấu hình exp001–010, implementation liên quan và các review nghiên cứu cũ. Không chạy notebook/model cũ hoặc đọc lại raw WAV. Đã xác minh hash các artifact baseline được pin; xác minh lại các bundle exp007/008 local; tính lại metric từ 16 CSV dự đoán người dùng tải về cho exp004–007 và so với frozen grid.

65 hàng kết quả cấu hình huấn luyện được tổng hợp từ baseline/exp001–008 và exp010. Exp009 là diagnostic gộp dự đoán, không phải một lượt training mới. Nguồn có ba mức khác nhau:

- **Artifact local đã đối chiếu hash:** baseline, exp007 và exp008. Đây là đối chiếu artifact lưu, không phải tái chạy inference hoặc kiểm tra trực tiếp Drive hiện tại.
- **Output notebook/review đã lưu:** exp002–006 và exp010; một số model exp004–007 có CSV dự đoán để tính lại chính xác. Bảng output thường làm tròn sáu chữ số.
- **Diagnostic exp009 đã lưu:** có bản sao local và output notebook xác nhận lưu/đọc lại; khác thời lượng quan sát với baseline một window. Đường dẫn bản sao có chữ `fixture_storage`, vì vậy không gọi đây là một lượt thực nghiệm độc lập.

Exp010 **đã có kết quả** trong output 04D hiện tại, không còn chỉ là cấu hình dự kiến: DS-CNN/03A window F1 55,35%, source F1 68,50%, selected epoch 13. Chưa có bundle artifact đầy đủ exp010 local để đối chiếu hash từng run như exp007/008.

Các bảng nhỏ nằm trong [results/B/previous_project_review](results/B/previous_project_review/):

- `experiment_metrics.csv`: 65 hàng, tách rõ validation window/source, train-eval, epoch, nguồn và mức bằng chứng.
- `experiment_overview.csv`: model tốt nhất theo validation window F1 ở mỗi nhóm, không trộn với source/test.
- `matched_deltas.csv`: so từng model/frontend với đúng đối chứng của thay đổi.
- `prediction_recomputation.csv`: metric tính lại từ 16 CSV, kiểm tra UID, parent metadata, nhãn, xác suất và argmax.
- `exp009_aggregation.csv`: ba endpoint gộp window; `verification.csv`, `sources.csv` và `notebooks_inspected.csv` giữ phạm vi/hash truy vết.

### Kết quả cần nhớ

Toàn bộ điểm dưới đây là **validation**, chưa phải kết quả trên A_v1. 03A cũ là log-mel; 03B cũ là Edge MFE, không phải notebook 03A chuẩn bị dữ liệu của rebuild.

| Thử nghiệm cũ | Kết quả quan sát | Bài học áp dụng |
| --- | --- | --- |
| Control exp002 | CNN-LSTM/03A có window F1 cao nhất 54,82%; DS-CNN/03A 54,10% | DS-CNN không thắng mọi recipe từ đầu; vẫn giữ các đối chứng |
| Exp003: giảm LR, chưa chuẩn hóa | Chỉ 4/8 cấu hình tăng window F1 | LR thấp hơn không phải cải tiến mặc định cho mọi model |
| Exp004: thêm TRAIN-global z-score | 6/8 tăng so control; TC-ResNet8/03A 40,64% → 60,38% | Cách chuẩn hóa có thể ảnh hưởng lớn hơn đổi model; đây là bằng chứng cho global, không phải per-bin |
| Exp005: giảm LR trên nền global norm | Chỉ 3/8 tăng; TC/03A 60,38% → 56,04% | So thay đổi trên cùng nền; không bê LR thấp sang mọi baseline |
| Exp006: class-weight exponent 1 → 0,5 | 7/8 tăng F1 tổng, nhưng Squ recall giảm ở cả tám | Macro-F1 tăng vẫn có thể bỏ sót lớp ít hơn; phải đọc precision/recall/support |
| Exp007: exponent 0,5 → 0,75 | Chỉ 4/8 tăng; DS/03A đạt 60,97% window F1, 74,89% source F1 | Đây là ứng viên phát triển cũ; không mặc định exponent 0,75 tối ưu cho dữ liệu mới |
| Exp008: Adam weight_decay 0 → 1e-4 | Chỉ 3/8 tăng; DS/03A 60,97% → 59,90% | Regularization không mặc định giúp; gap nhỏ hơn không tự đồng nghĩa validation tốt hơn |
| Exp009: gộp hai window | Matched single 61,32% → pair 64,90%, tăng 3,58 điểm; support 1,935 giây | Gộp nhiều quan sát hữu ích ở tập này nhưng thay đổi thời lượng và đơn vị quyết định; không gọi là model một window đạt 64,90% |
| Exp010: time mask nhẹ | DS/03A 60,97% → 55,35% window F1 | Kết quả âm cần giữ lại; augmentation chỉ là giả thuyết để kiểm chứng |

Các delta nhỏ từ một seed và validation đã tuning nhiều lần không chứng minh cải thiện ổn định. Mean của tám cấu hình không phải điểm ensemble.

### Các bài học cụ thể phải dùng trong B–F

1. **Chuẩn hóa:** recipe thành công cũ dùng một scalar mean/std cho toàn frontend, fit TRAIN. B mới đang dùng mean/std theo từng mel bin cho nhánh log-mel. Hai cách khác nhau; chưa có bằng chứng từ bài cũ cho việc per-bin tốt hơn. Giữ code B đã duyệt, nhưng ưu tiên đối chứng none/global/per-bin ở C trước khi kết luận ảnh hưởng kiến trúc. Không âm thầm đổi baseline hoặc gán kết quả global cũ cho per-bin mới.
2. **Trọng số lớp:** DS/03A Squ recall exp004/006/007 là 67,80% / 37,29% / 48,81%, trong khi window F1 tổng là 56,16% / 59,92% / 60,97%. Cần giữ theo dõi lớp ít ở project mới và precision/recall cùng lúc. B hiện không weight là mốc khởi đầu; C mới so unweighted/soft weights/sampling trên cùng pool, không sao chép bốn trọng số của loài cũ.
3. **Cấp dự đoán:** 74,89% là source F1, không phải window F1. B mới chọn theo clip F1; clip/source/event bẫy không được coi là cùng một đơn vị. Mọi bảng phải ghi rõ horizon và support.
4. **Gộp window:** đề xuất C so 1/2/4 window trên cùng population và thời lượng đã khai báo, giữ trong cùng clip. Với 300 ms/hop 150 ms, hai window liên tiếp phủ 450 ms, bốn phủ 750 ms. Chưa triển khai hoặc thay metric chính của A_v1. Full-clip aggregation hiện có không chứng minh chất lượng một sự kiện bay qua cổng.
5. **Augmentation theo tỷ lệ:** time mask 1–8 frame ở 96 frame cũ không được chép nguyên sang 27 frame mới (8 frame là gần 30% chiều thời gian mới). Exp010 không giúp ở trial đã thấy. Noise/gain/masking phải có đối chứng, đánh giá per-class và giữ augmentation ngoài loader eval/validation.
6. **Frontend:** log-mel 03A cũ hơn Edge MFE 03B ở cả bốn model trong exp007/008, nhưng đó là hai pipeline khác nhiều thành phần, không cô lập tác động riêng một phép toán. Không kết luận mọi MFE kém hoặc log-mel mới tương đương VGGish/Edge Impulse.
7. **Sàn tín hiệu/geometry:** audit Edge MFE cũ chọn −70 dB sau kiểm tra TRAIN; so −65/−70 có 11/0 tensor toàn zero trên 22.613 window. Log offset 0,01 cũ, dB/noise floor MFE và power floor 1e-12 mới khác đơn vị. Không chép ngưỡng, filter bank hoặc shape 16 kHz sang 8 kHz; giữ kiểm tra tone/silence/gain và frame/sample đầy đủ. Những số đo audit cũ là bằng chứng đã lưu, không phải quét lại raw data hôm nay.
8. **Âm thanh nhỏ/lỗi lặp:** audit 43 WAV/484 window TRAIN cũ không có zero/clipping số trong mẫu; sai có cả tín hiệu yếu lẫn mạnh. Không xóa clip, relabel hoặc tăng gain tự động chỉ vì RMS thấp, confidence cao hay model liên tục sai.
9. **Nguồn/điều kiện thu:** lỗi cũ khác nhau mạnh theo loài × phương pháp bắt (HBN/LT); đây không phải nhãn microphone và chưa chứng minh nguyên nhân. Source-name split cũ cũng chưa tách độc lập phiên thu (review ghi 41 ngày có ở cả ba tập). A_v1 mới tiếp tục công bố nguồn chưa xác minh và validation albopictus một nguồn; không gọi nhóm tạm là cá thể độc lập.
10. **Overfit và stopping:** giữ so sánh train-eval/validation của cùng checkpoint; online train loss/F1 có dropout và trọng số thay đổi nên không dùng thay cho train-eval gap. Tăng epoch/model size không tự giải quyết plateau. B mới chọn và dừng theo clip F1, khác stopping theo weighted loss cũ; ghi nhận đây là khác biệt protocol.
11. **Bất định:** bootstrap theo nguồn trong review cũ chỉ mô tả biến thiên từ những prediction cố định, không thay việc chạy nhiều training seed, và không xác nhận độc lập cá thể/session. Một nguồn albopictus validation không thể tạo thêm đa dạng sinh học bằng bootstrap hay seed.
12. **Chi phí:** DS-CNN 6.244 tham số là lý do giữ ưu tiên; không phải chứng nhận nhanh nhất/ít RAM nhất trên ESP32. Checkpoint PyTorch, weight bytes lý thuyết, MAC và arena RAM khác nhau; cả frontend và inference phải được đo khi tích hợp.
13. **Workflow:** giữ code trên Git và data/cache/kết quả lớn trên Drive khi chuyển Colab; xác minh source/cache và restart kernel sau đổi module. Giữ run identity, checkpoint, dự đoán và report theo từng thử nghiệm; không ghi đè, nhầm output cũ hoặc trộn smoke với full. Giữ notebook hiện tại đơn giản, không bê toàn bộ hệ thống nhiều engine/config cũ sang rebuild.

### Quyết định sau lần rà này

- Đã lưu bài học và số liệu vào context project; không chỉ dựa vào lời nhớ rằng DS-CNN từng thắng.
- Giữ A_v1, code B, thư viện và kết quả smoke hiện tại nguyên byte. Không chạy model cũ/mới, không khởi chạy full, không mở test của A_v1 hoặc đổi recipe trong lượt review.
- Khi thiết kế C hoặc đề xuất thay B, phải đối chiếu mục này và bằng chứng, ghi rõ câu hỏi, đối chứng và thay đổi file để người dùng duyệt trước khi code. Ưu tiên kiểm tra normalization/imbalance/giới hạn nguồn, thay vì mặc định thêm độ phức tạp hoặc chép cấu hình từng thắng.


## 13. Bổ sung Enhanced MicroDSC-style đã được duyệt — 20/10/2026

Người dùng đồng ý thêm baseline thứ tám. Phạm vi thực hiện vẫn là **code + kiểm tra nhỏ**, không tự khởi chạy full hoặc test. Sửa notebook 04, README và kế hoạch này; cập nhật report/bảng/hình/model trong results/B/smoke. Không thay A_v1, frontend, các baseline trước, môi trường thư viện hoặc project cũ.

- Nguồn: Kazeneza và các đồng tác giả (2026), [paper](https://doi.org/10.7717/peerj-cs.3925), [code v1.0.0 / CC BY 4.0](https://doi.org/10.5281/zenodo.18879243). Pinned training notebook commit `8603867`, SHA256 `cdf303981d12f057ed27bc1e9ee00ed7c4e14aa5abb78232633239eabff2e7cb`. Ghi công trong notebook và report.
- Vai trò: so sánh với DS-CNN hiện tại trên cùng log-mel/split/recipe, kiểm tra tác động tổng thể của phân bố kênh, tổ chức block, ReLU6 và pooling. Không nhập thêm CNN lớn/Standard DSC của paper. Chưa cô lập từng thành phần; việc đó dành cho C.
- Thiết kế: stem 16, ba block depthwise/pointwise 32/48/64, không bias convolution, ReLU6 sau depthwise và sau pointwise/BN; ba pool 2×2 VALID, GAP, dropout 0,2, head bốn logits. BN epsilon 0,001/momentum PyTorch 0,01 để tương ứng mặc định Keras; khởi tạo trọng số vẫn theo PyTorch. Không bật residual.
- Input: 40×27 từ 300 ms/8 kHz theo B, khác 5 giây/16 kHz/MFE của paper. Stem SAME bất đối xứng theo tần số; kích thước 40×27 → 20×14 → 10×7 → 5×3 → 2×1 → GAP 1×1. Không pad audio hoặc resize ảnh. Chưa biết pooling có làm mất thông tin hữu ích; giữ làm câu hỏi validation/C, không âm thầm đổi kiến trúc.
- Đếm tham số: bản bốn lớp có 6.708 trainable parameters và 320 phần tử moving mean/variance BN; tổng theo cách đếm Keras là 7.028. Đổi phép đếm sang head 11 lớp cho 7.483, khớp reference. Không coi 7.483 là số trainable của bản bốn lớp; không cộng integer num_batches_tracked vào đối chiếu Keras.
- Report cần chứa đủ tám model + majority, bảy đường học neural, cost/latency/checkpoint và bảng shape/count/source MicroDSC. Giữ nguyên số mẫu smoke và ngân sách hai epoch × hai batch mỗi neural model. Số đo CPU không đại diện ESP32-S3; chưa thêm INT8.
- Khi thiết kế E/phần cứng: tham khảo pipeline lượng tử hóa và đo frontend + inference + RAM trên board. Latency 0,17 ms trong phần đánh giá paper được đo trên Intel i7, không gán cho ESP32-S3; không bê ngưỡng confidence hoặc cơ chế gộp nhiều detection vào bài muỗi khi chưa có kiểm chứng.

Trạng thái thực thi: **đã hoàn tất code và smoke**. Notebook có đủ tám baseline + majority; bảy neural model mỗi model hai epoch/bốn update. 28 kiểm tra PASS, bảy checkpoint nạp lại khớp xác suất; MicroDSC có 6.708 trainable parameters, 26.832 byte trọng số FP32 và khoảng 404.496 MAC cho input 40×27 (chưa gồm frontend/BN/activation/pooling). Đã mở report bằng trình duyệt headless: đủ sáu hình, không lỗi JavaScript hoặc tràn ngang; đã xem hình confusion đủ chín panel. 71 artifact được kiểm hash. Không có kết quả full và không đọc/dự đoán audio test. Kết quả smoke không dùng xếp hạng model.

Bảy baseline trước cùng majority giữ nguyên dự đoán và metric so với smoke trước khi bổ sung; 140 file A/data metadata/manifests/review project cũ được đối chiếu hash không đổi. Đã bổ sung thử lưu lại checkpoint nguyên tử khi Windows khóa file tạm thời (tối đa sáu lần); kiểm tra lỗi tạm phục hồi, lỗi kéo dài vẫn báo lỗi và giữ checkpoint trước đó. Không thay optimizer, seed, sampling hoặc frontend. Bản smoke trước được sao lưu ngoài project vào thư mục TEMP của Windows để phục hồi khi cần; không tạo thêm bản dữ liệu hay folder archive trong workspace.


## 14. Đóng gói dữ liệu và repo riêng — 20/10/2026

Đã được yêu cầu tạo gói Drive và làm sạch/push repo riêng `ancaranovik/Edge_AI_Mosquito`. Gói `drive_upload/Edge_AI_Mosquito/A_v1_bundle.zip` gồm 2.905 WAV của pool A_v1, manifests/metadata gốc, kết quả A, B smoke, bảng review project cũ và bản notebook đã chạy. Không chứa `.venv`, feature cache hoặc toàn bộ raw HumBugDB/BioDCASE. Giữ nguyên byte dữ liệu và manifest; SHA256/CRC của toàn bộ 3.118 file đã kiểm tra sau nén. Gói có inventory bên trong và checksum archive bên ngoài.

Notebook trong working tree/Git đã bỏ output và execution count, giữ nguyên từng code/Markdown cell. A notebook source hash vẫn khớp freeze; output gốc nằm trong archive tại `notebook_outputs/`. `.gitignore` loại data/results/upload/environment/model binary; `.gitattributes` giữ LF cho source. Không sửa project cũ hoặc chạy full/test.

Đây mới là đóng gói và quản lý source. Notebook B vẫn là local CPU; chuyển sang Colab cần triển khai mount, ánh xạ đường dẫn Windows/Linux, cài môi trường thích hợp và đường GPU/checkpoint/đo thời gian trước khi chạy đầy đủ. Không coi việc push Git hoặc đổi kernel là đã hoàn tất bước này.
