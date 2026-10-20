# Mosquito classification

Chạy local bằng `.venv/Scripts/python.exe` (Python 3.12). Mở notebook trong VS Code,
chọn kernel `.venv`, rồi Run All. Khi cần cài lại môi trường:

```powershell
.venv/Scripts/python.exe -m pip install -r requirements.txt
```

```text
data/raw/               WAV HumBugDB gốc
data/metadata/          Metadata gốc
data/manifests/         Kiểm kê, nhãn, split và chỉ mục window
notebooks/             Phân tích và thí nghiệm
results/A/             Bảng, biểu đồ và report phần A
config.json            Đường dẫn dùng bởi notebook 01–02
```

## Phần A

- `01_A_data_analysis.ipynb`: khảo sát HumBugDB 0.0.2.
- `02_A_biodcase_analysis.ipynb`: kiểm kê BioDCASE, đối chiếu tín hiệu với HumBugDB.
- `03_A_data_preparation.ipynb`: chọn bốn loài, làm sạch, rà nguồn, chia tập và tạo chỉ mục window.

Phần A đã chốt ở A_v1 cho Ae. aegypti, Ae. albopictus, An. dirus và An. minimus.
Notebook 03 khai báo đường dẫn trực tiếp ở cell đầu, không đọc `config.json`.
WAV BioDCASE được đọc từ Downloads; không sao chép hoặc xuất WAV nhỏ hàng loạt.

Kết quả chính:

- `data/manifests/biodcase_four_species_clips.csv`
- `data/manifests/biodcase_four_species_windows.csv`
- `results/A/preparation/report.html`
- `results/A/preparation/tables/dataset_freeze.csv`

Dùng cột `split` cho nghiên cứu; `reference_split` giữ phân vùng tham chiếu.
A_v1 giữ domain test theo từng loài, không phải domain mới đối với toàn bộ training.
Đã tách nhóm nguồn/tín hiệu liên quan được phát hiện; chưa xác minh độc lập cá thể
cho mọi clip. Validation albopictus chỉ có một nguồn D1; training là D5, test là D2.
Loại cảm biến của D5 chưa xác minh. Kết quả chưa chứng minh khả năng dùng INMP441.
Chi tiết và quyết định từng loài ở report và `split_design_review.csv`.

## Phần B–F

B baseline → C ablation → D kết hợp → E dò tham số → F xác nhận nhiều seed và test cuối.
B bắt đầu bằng kiểm tra checksum của bản A_v1, phát triển/chọn model trên train và
validation. B–E chưa dự đoán/chấm điểm test. F đánh giá test sau khi chốt phương pháp
và tham số. Các biến đổi học từ dữ liệu chỉ fit trên training.

B có kế hoạch tám baseline: Small CNN, DS-CNN, TC-ResNet8, CNN-LSTM,
đặc trưng phổ + Random Forest, CNN 1D theo MosquitoSong+, BC-ResNet-1 và
Enhanced MicroDSC-style (đã duyệt bổ sung; dùng chung log-mel 40×27).
Phạm vi, thông số dự kiến và file sẽ tạo/sửa nằm trong [B_PLAN.md](B_PLAN.md).
Đã duyệt phạm vi **code + cài phụ thuộc + kiểm tra nhỏ** ngày 20/10/2026.
Notebook [04_B_baseline.ipynb](notebooks/04_B_baseline.ipynb) mặc định
`RUN_MODE='smoke'`: kiểm tra bằng ít clip train/validation, chưa chạy đầy đủ.
Mở notebook, chọn kernel `.venv`, rồi Run All. PyTorch được pin bản CPU
trong requirements; không cần cài librosa, torchaudio hoặc torchvision.

Đầu ra smoke: `results/B/smoke/report.html`, cùng `tables/`, `figures/`,
`models/`. `results/B/report.html` dẫn tới các chế độ đã chạy; cache ở
`results/B/cache/`. Điểm smoke chỉ để kiểm tra code, không xếp hạng model.
Huấn luyện đầy đủ chưa được khởi chạy. Khi chủ động muốn chạy đầy đủ,
đổi `RUN_MODE='full'` trong notebook; đầu ra lưu riêng ở `results/B/full/`.
Báo cáo metric cấp clip, recall từng loài, tham số và kích thước trọng số.
Độ trễ/RAM thực tế trên ESP32-S3 cần được đo ở giai đoạn tích hợp thiết bị.

MicroDSC-style tham khảo [Kazeneza et al. (2026)](https://doi.org/10.7717/peerj-cs.3925)
và [code v1.0.0](https://doi.org/10.5281/zenodo.18879243): kênh 16/32/48/64,
ReLU6, ba pool và GAP. Đây là bản thích nghi cho 300 ms và bốn loài; report có
bảng kích thước từng stage và cách đếm tham số, không gán điểm/latency paper cho ESP32.

Đã kiểm tra smoke cho đủ tám baseline: 28 kiểm tra PASS; MicroDSC-style có
6.708 tham số học được. Chi tiết xem `results/B/smoke/report.html`; chưa chạy full.

## Git và dữ liệu Drive

Repo: https://github.com/ancaranovik/Edge_AI_Mosquito

Git chỉ lưu notebook, cấu hình và tài liệu. `data/`, `results/`, `.venv/` và
`drive_upload/` không đưa lên Git. Notebook trên Git đã bỏ output nhúng;
code và Markdown giữ nguyên, không thay protocol A_v1.

Folder để upload lên Drive B: `drive_upload/Edge_AI_Mosquito/` gồm:

```text
A_v1_bundle.zip    2.905 WAV của pool bốn loài, manifests, metadata và kết quả đã lưu
SHA256SUMS.txt     SHA256 của archive
UPLOAD.txt        Cách dùng gói dữ liệu
```

Archive giữ đúng cấu trúc `data/` và `results/` để giải nén vào thư mục repo
trên máy Colab. `bundle_inventory.csv` bên trong ghi hash từng file.
`notebook_outputs/` giữ bản notebook đã chạy trước khi làm sạch output Git;
không chép các bản này đè lên notebook mới trong repo.

Đây là gói A_v1 bốn loài, không phải toàn bộ HumBugDB/BioDCASE gốc. Audio
được lưu ở `data/audio/biodcase_A_v1/`; manifest gốc giữ nguyên đường dẫn
Windows để bảo toàn checksum. Notebook B hiện dùng cấu hình **local CPU**.
Trước khi chạy Colab cần thêm mount Drive, ánh xạ đường dẫn và cấu hình
GPU/môi trường; chỉ đổi kernel hoặc `RUN_MODE` chưa đủ. Chưa khởi chạy full.
