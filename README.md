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

Đầu ra mới: `results/B/baseline_v1/smoke/report.html`, cùng `tables/`, `figures/`,
`models/`. Cache ở `results/B/baseline_v1/cache/`. Kết quả smoke trước khi chuyển
Colab vẫn giữ tại `results/B/smoke/`, không ghi đè. Điểm smoke chỉ để kiểm tra code, không xếp hạng model.
Huấn luyện đầy đủ chưa được khởi chạy. Khi chủ động muốn chạy đầy đủ,
đổi `RUN_MODE='full'` trong cell đầu; đầu ra lưu riêng ở `results/B/baseline_v1/full/`.
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
Windows để bảo toàn checksum. Notebook B ánh xạ các tên WAV sang folder trên
Colab mà không viết lại manifest. Bản Colab đã được bổ sung ngày 21/10/2026;
archive cũ tiếp tục dùng được, không cần upload lại. Chưa khởi chạy full.


## Chạy B trên Colab từ VS Code

1. Pull code mới từ `main` về máy, mở `notebooks/04_B_baseline.ipynb`.
2. Chọn kernel Colab của tài khoản A, chọn runtime có GPU.
3. Shortcut Drive mặc định: `/content/drive/MyDrive/Edge_AI_Mosquito`.
   Folder chứa `A_v1_bundle.zip`; A phải đọc được data và ghi được folder kết quả.
4. Giữ `RUN_MODE='smoke'`, chạy các cell mục 0 rồi Run All. Cell setup mount
   Drive A, clone/pull repo trên runtime, copy/kiểm checksum archive, giải nén
   vào ổ tạm. Không chép lại output B/notebook cũ trong archive lên code mới.
5. Colab giữ PyTorch CUDA có sẵn, chỉ cài thư viện thiếu; không dùng
   `pip install -r requirements.txt` bản CPU trong Colab. Local vẫn dùng file đó.
6. Kiểm tra phần device in ra và report smoke trên runtime. Chỉ khi xác nhận
   smoke ổn mới chủ động đổi `RUN_MODE='full'` và Run All.

Các tham số người dùng sửa trực tiếp ở cell đầu. Những cell thuật toán được
đối chiếu với source trên kernel để phát hiện notebook VS Code và code đã pull
khác nhau. Nếu sửa thuật toán thì save/push code, pull lại và mở bản notebook
mới; chỉ đổi giá trị ở cell thiết lập không cần commit riêng.

RF/frontend dùng CPU. Neural model dùng GPU khi khả dụng; Colab mặc định dừng
nếu không có GPU. Recipe B vẫn float32, không AMP/TF32; seed/early stopping,
frontend, split và tám kiến trúc giữ như đã chốt. Report luôn đo bản sao neural
trên CPU để so chi phí với RF, có bảng GPU riêng khi chạy CUDA; GPU timing có
synchronize, không tính copy input CPU→GPU hoặc thu âm.

Checkpoint được lưu sau mỗi epoch về:

```text
Edge_AI_Mosquito/runs/B/baseline_v1/<smoke|full>/<run_id>/
  <model>/slot0.zip, slot1.zip, slot0.csv, slot1.csv
  report/slot0.zip hoặc slot1.zip
  report.html
```

Hai snapshot luân phiên cho phép chọn bản trước còn hợp lệ nếu upload mới bị
ngắt. Khi runtime mới, Run All với cùng code/cấu hình/phiên bản/device để dựng
lại data/cache và khôi phục từ epoch đã lưu. Epoch dở chạy lại. Không tự ghép
checkpoint khác GPU/recipe/phiên bản; đổi cấu hình thì đổi `EXPERIMENT_NAME`
để giữ riêng kết quả. Nếu lỗi lưu Drive, notebook dừng; kiểm tra quyền Editor
và quota của A. A tạo file mới trong folder B chia sẻ vẫn có thể dùng quota A.

Notebook không tự chạy full/test trong bước chuẩn bị này. Kiểm tra local CPU
và filesystem giả lập không thay thế smoke CUDA/Drive thật trên Colab.


Kiểm tra bản chuyển Colab ngày 21/10/2026: 32 smoke checks và 21 integration
checks đã PASS trên local CPU/Drive giả lập. Bằng chứng ở
`results/B/colab_persist_smoke/`; chưa phải kết quả CUDA/Drive thật hoặc full.
`source_notebook.ipynb` trong report là source repo trên kernel; các giá trị
thiết lập thực tế (kể cả khi sửa cell đầu) nằm trong `tables/run_settings.csv`.
