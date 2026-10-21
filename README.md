# Mosquito classification — ESP32-S3 + INMP441

## Notebook

```text
notebooks/
  01_A_data_analysis.ipynb                 Khảo sát HumBugDB, dùng chung
  02_A_biodcase_analysis.ipynb             Khảo sát BioDCASE, dùng chung
  A_v1/
    03_A_data_preparation.ipynb            Albopictus train D5, validation D1
    04_B_baseline.ipynb                    Baseline A_v1, giữ output đã chạy
    05_B_domain_diagnostics.ipynb           Chẩn đoán khác biệt nguồn D5–D1
    06_B_controlled_experiments.ipynb       Đối chứng nguyên nhân, giữ output full
  A_v2/
    03_A_data_preparation.ipynb            Chia D5 train/validation, report và freeze mới
    04_B_baseline.ipynb                    Tám baseline từ đầu trên A_v2
```

01–02 không đổi. Notebook A_v1 được chuyển folder và cập nhật đường dẫn;
output/execution count đã có được giữ nguyên như bằng chứng lịch sử. Freeze,
manifest, kết quả A_v1 và archive cũ không thay đổi. Verifier cho phép đúng
thay đổi vị trí A03 khi đối chiếu source hash lịch sử. Run ID của code sau
chuyển folder có thể khác; không bỏ kiểm tra source/run ID để tiếp tục
checkpoint cũ. Source trước chuyển folder đã sao lưu ngoài repo trong TEMP.

## Protocol A_v2

Bốn nhãn: 0 aegypti, 1 albopictus, 2 dirus, 3 minimus. Mono 8 kHz,
window 300 ms/hop 150 ms; giữ pool audio và QC đã chốt ở A_v1.

- Albopictus D5: 1.099 clip train / 275 clip validation, seed 42,
  chọn khoảng 20% nhóm cho validation. Chia nhóm trước khi gán window.
- Albopictus D1: `diagnostic`, 79 clip giữ lại nhưng chỉ 67 clip đủ window
  (164 window). Không dùng cho train, normalization, early stopping hoặc chọn model.
- Test và split ba loài còn lại giữ nguyên A_v1. B–E chưa dùng test để phát triển.
- D5 thiếu mapping phiên thu/cá thể và chưa xác minh loại cảm biến. Nhóm
  `unverified_clip` không phải nguồn độc lập đã xác minh. D5 chỉ gắn với
  albopictus nên model có thể học đặc điểm domain. Điểm cao A_v2 chưa chứng
  minh chuyển nguồn hoặc hoạt động trên INMP441.

Kết quả chuẩn bị: `results/A_v2/preparation/report.html` và
`data/manifests/A_v2/`. Kết quả B mới: `results/A_v2/B/baseline/`.
Audio dùng chung từ `data/audio/biodcase_A_v1/` hoặc raw Downloads trên local;
không nhân đôi WAV. A_v2/03 kế thừa audit/QC/nhóm gần trùng A_v1, không tìm
kiếm gần trùng mới hoặc chứng nhận độc lập cá thể.

B baseline → C thử từng thay đổi → D kết hợp → E dò tham số → F xác nhận
nhiều seed và test cuối sau khi cố định phương pháp. 05–06 của A_v1 là
chẩn đoán thất bại chuyển nguồn, không thay thế C–E của A_v2.

## Drive và Colab

Gói bổ sung nằm tại `drive_upload/Edge_AI_Mosquito/A_v2_update/`:

```text
A_v2_metadata.zip     Manifest, report và freeze mới; không chứa WAV
SHA256SUMS.txt        Checksum gói bổ sung
UPLOAD.txt           Cách upload và chạy
```

1. Upload **folder `A_v2_update`** vào folder `Edge_AI_Mosquito` trên Drive B,
   đặt cạnh `A_v1_bundle.zip`. Notebook đọc `A_v2_update/A_v2_metadata.zip`;
   các file checksum/hướng dẫn v2 giữ trong folder đó. Không giải nén trên Drive.
2. Colab tài khoản A tiếp tục dùng shortcut
   `/content/drive/MyDrive/Edge_AI_Mosquito`.
3. Cập nhật code repo `https://github.com/ancaranovik/Edge_AI_Mosquito` trước
   khi chạy. Lượt chuẩn bị A_v2 này chưa tự commit/push.
4. Mở `notebooks/A_v2/04_B_baseline.ipynb`, giữ `RUN_MODE='smoke'`, chọn GPU
   Colab rồi Run All. Notebook kiểm SHA256/inventory cả hai gói và giải nén
   vào ổ tạm, dùng đúng freeze A_v2.
5. Smoke thành công mới đổi `RUN_MODE='full'` để huấn luyện từ đầu. Không
   nạp checkpoint, cache hoặc normalization A_v1 làm kết quả A_v2.

Kết quả Drive: `Edge_AI_Mosquito/runs/A_v2/B/baseline/<mode>/<run_id>/`.
RF/frontend chạy CPU; neural model dùng GPU Colab. Checkpoint vẫn có hai
snapshot luân phiên, checksum và RNG để tiếp tục khi mất runtime. Latency
Colab/CPU không thay phép đo thực tế trên ESP32-S3.

Muốn đổi tỷ lệ/seed chia A_v2: sửa cell đầu A_v2/03, chạy lại và kiểm report;
sau đó phải đóng gói lại metadata, cập nhật checksum ở cell đầu A_v2/04 rồi
upload. Không sửa CSV đơn lẻ rồi bỏ qua freeze.

## Local

Chọn kernel `.venv`, chạy notebook từ repo. Nếu cần cài môi trường:

```powershell
.venv/Scripts/python.exe -m pip install -r requirements.txt
```

`config.json` chỉ dùng cho 01–02; notebook chuẩn bị/huấn luyện khai báo tham
số trực tiếp. Git chỉ chứa source/tài liệu; data, results, môi trường và gói
upload được `.gitignore` loại trừ.
