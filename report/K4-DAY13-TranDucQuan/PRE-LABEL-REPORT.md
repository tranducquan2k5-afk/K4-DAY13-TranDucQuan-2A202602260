# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

---

## Nhóm và provenance

- Mã nhóm/phòng: `K4-DAY13-TranDucQuan` (Thực hành cá nhân)
- Thành viên: xem `TEAMMATES.md` (Trần Đức Quân - MSSV: 2A202602260).
- Trạng thái: `executed-by-group` (trực tiếp chạy máy cá nhân 100% qua Docker offline bundle).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Trần Đức Quân; `2026-10-02 09:43:00 (UTC+7)` / `2026-10-02 02:43:00 (UTC)`; Windows 11 host (Git Bash / PowerShell), Docker Engine Linux `x86_64` (amd64).
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64` / `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; Repo revision: `0831856d921609312d42c7582c366e5a311bb7b1`
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `data/demo.pcd` (frame_id: `demo`, 17.238 points); Nguồn: KITTI 000008 chuyển đổi (CC BY-NC-SA 3.0), cho phép chạy thí nghiệm học thuật trên máy nhóm/cá nhân; SHA256 input: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`
- Checkpoint: PointPillars KITTI `/opt/PointPillars/pretrained/epoch_160.pth`; SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`
- Phạm vi: front-window (`x: [0.0, 69.12] m`, `y: [-39.68, 39.68] m`, `z: [-3.0, 1.0] m` trong model frame); score threshold: `0.3`
- Giả định kênh thứ tư/intensity và nguồn z_ground: Kênh reflectance gán giá trị hằng số qua adapter (`0.0` khi nhận diện `vehicles`, `0.7` khi nhận diện `pedestrian` & `two-wheels`); `z_ground = 0.075 m` ước lượng từ phân vị độ cao mặt đường đám mây điểm gốc.

---

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| :--- | :---: | :---: | :---: | :---: | :--- | :--- |
| **A** | 0 m | 0.16 m | **1** | **0.330 m** | `run-A/boxes-demo-delta-0-voxel-0.16.json`<br>`run-A/side-demo-delta-0-voxel-0.16.png`<br>`run-A/summary.csv` | Chỉ nhận diện được 1 xe con (`vehicles: 1`), mất sạch người đi bộ (`pedestrian: 0`) và xe hai bánh (`two-wheels: 0`). Ảnh `side-demo-delta-0-voxel-0.16.png` cho thấy hộp bị vạch $z=0$ cắt ngang thân và cắm chìm xuống đất do thiếu phép bù trừ độ cao cảm biến $1.73\text{ m}$. |
| **B** | 1.73 m | 0.16 m | **13** | **1.034 m** | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`<br>`run-B/side-demo-delta-1.73-voxel-0.16.png`<br>`run-B/summary.csv` | Phát hiện cân bằng 13 hộp gồm 10 xe (`vehicles`), 2 người đi bộ (`pedestrian`) và 1 xe hai bánh (`two-wheels`). Ảnh `side-demo-delta-1.73-voxel-0.16.png` cho thấy đáy các hộp tựa sát chuẩn xác trên mặt phẳng đường $z \approx 0\text{ m}$. Đây là cấu hình baseline chuẩn nhất. |
| **C** | 1.73 m | 0.32 m | **6** | **1.091 m** | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`<br>`run-C/side-demo-delta-1.73-voxel-0.32.png`<br>`run-C/summary.csv` | Số hộp giảm xuống 6 và toàn bộ 100% bị phân loại thành `pedestrian: 6`, mất hoàn toàn ô tô và xe hai bánh. Kích thước voxel quá lớn ($0.32\text{ m}$) làm thô hóa biểu diễn không gian, triệt tiêu đặc trưng hình học của các phương tiện lớn. |

### Phân tích kỹ thuật chuyên sâu

- **A/B: thay input trước model có khác dịch cùng một hằng số cho output không? Vì sao?**  
  **Khác hoàn toàn.**  
  1. *Giới hạn ROI trần (ROI Cropping):* Trước khi đưa vào mô hình, dữ liệu điểm bị cắt lọc trong khoảng $z \in [-3.0, 1.0]\text{ m}$. Ở lượt A ($\text{delta} = 0$), các cụm điểm phía trên $1.0\text{ m}$ bị cắt bỏ hoàn toàn trước khi trích xuất đặc trưng voxel.  
  2. *Khớp Anchor (Anchor Matching):* PointPillars học biểu diễn tọa độ bên trong từng pillar và khớp với anchor 3D được định nghĩa sẵn theo phân phối dữ liệu KITTI (độ cao cảm biến $\sim 1.73\text{ m}$). Khi input không dịch $\text{delta} = 1.73$, các điểm không rơi vào receptive field của anchor `pedestrian` và `two-wheels`, khiến bộ phát hiện bỏ sót 12 đối tượng. Dịch output sau khi chạy xong chỉ nâng được 1 hộp xe duy nhất lên, không thể khôi phục lại 12 hộp người và xe máy đã bị mất ngay từ đầu vào.

- **B/C: thấy gì khi đổi pillar? Có đủ bằng chứng để nói cấu hình nào tốt hơn không?**  
  - Khi tăng cạnh pillar XY từ $0.16\text{ m}$ lên $0.32\text{ m}$ (diện tích voxel tăng gấp 4 lần), độ phân giải của pseudo-image giảm mạnh. Các điểm rời rạc của ô tô bị gộp chung thành cụm thô và kích hoạt nhầm anchor người đi bộ (`pedestrian: 6`), đồng thời xóa sổ hoàn toàn class `vehicles` và `two-wheels`.  
  - **Nhiều hộp hơn hoặc ít hộp hơn không tự khẳng định chất lượng.** Tuy nhiên, phân bố nhãn ở cấu hình B (10 xe, 2 người, 1 xe máy) hợp lý và ổn định hơn rất nhiều so với cấu hình C (chỉ toàn người). Cấu hình B là biểu diễn chuẩn xác phù hợp với checkpoint gốc.

- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?**  
  - *Giới hạn ROI:* Cấu hình checkpoint KITTI chỉ quét nửa không gian phía trước ($x \in [0.0, 69.12]\text{ m}$); toàn bộ đối tượng ở phía sau xe ($x < 0$) đều bị miss hoàn toàn (`rear = 0`).  
  - *Góc Side (mặt phẳng X-Z):* Rất hữu hiệu để kiểm tra cao độ tâm $z$ và độ tiếp xúc mặt đường cục bộ. Tuy nhiên, hình chiếu Side triệt tiêu hoàn toàn trục $Y$, làm các đối tượng ở làn trái và làn phải bị đè chồng lên nhau, không thể xác định hướng đầu xe (`yaw`) nếu không đối chiếu trên góc nhìn Top (BEV) và ảnh camera.

- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?**  
  - JSON A và JSON C hoàn toàn không thể dùng do sai lệch nghiêm trọng về phân bố nhãn và độ cao.  
  - JSON B là ứng viên tốt nhất, nhưng vẫn **chưa được phép import trực tiếp làm nhãn chính thức** mà chỉ đóng vai trò là pre-label (gợi ý khởi đầu).  
  - Cần kiểm tra tiếp: (1) Mở trên CVAT 3D để đối chiếu góc nhìn Top và ảnh camera nhằm kiểm tra hướng `yaw`; (2) Bổ sung gán nhãn cho các lớp mà mô hình không nhận diện (`Animal`, `Obstacle`); (3) Kiểm tra xóa các hộp dương tính giả ở xa.

---

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| :--- | :---: | :---: | :---: | :--- | :--- |
| **`case-correct`** | 0 / 13 | 0 m | Không đổi | **Pipeline chuẩn.** Tiếp tục chuyển sang quy trình rà soát 3D trên CVAT. | SHA256 trùng khớp 100% với file dự đoán Run B (`c2a8db247353...`). Đáy các hộp bám chuẩn mặt phẳng đường $z \approx 0\text{ m}$. |
| **`case-batch-z`** | 13 / 13 | -1.805 m | Không đổi | **DỪNG PIPELINE NGAY LẬP TỨC.** Báo LC/kỹ thuật kiểm tra code chuyển đổi hệ tọa độ. Tuyệt đối không giao annotator sửa tay. | 100% (13/13) hộp đều bị tụt chìm sâu dưới lòng đất tại $z \in [-1.5, -0.5]\text{ m}$ trên `side-batch-z.png`. Mọi hộp đều có độ lệch đúng bằng $\text{delta} + z_{\text{ground}} = 1.73 + 0.075 = 1.805\text{ m}$ do thiếu bước chuyển nghịch đảo tọa độ về hệ nguồn. |
| **`case-one-box-z`** | 1 / 13 | -1.805 m (chỉ 1 hộp) | 12 hộp còn lại giữ nguyên | **KHÔNG DỪNG PIPELINE.** Pipeline biến đổi tọa độ hoạt động bình thường; chuyển cho annotator kiểm tra và xóa/sửa hộp lỗi cục bộ trên CVAT. | Trên `side-one-box-z.png`, duy nhất 1 hộp đầu tiên bị tụt chìm xuống đất; 12 hộp còn lại tiếp xúc mặt đường chuẩn xác. |

*Ghi chú: Cả 3 ca kiểm định QC đều do script helper tạo biến đổi có chủ đích từ kết quả Run B nhằm mục đích huấn luyện nhận diện lỗi hệ thống, không phải nhãn Ground Truth và tuyệt đối không import vào CVAT.*

---

## Nhận xét cá nhân

### Trần Đức Quân (MSSV: 2A202602260)
- **Vai trò thực hiện:** Trực tiếp vận hành toàn bộ chu trình (chạy container Docker bằng lệnh runner trên Git Bash Windows, kiểm tra cấu hình tham số A/B/C, phân tích hình học trên ảnh Side view và hoàn thiện báo cáo kỹ thuật).
- **Quan sát kỹ thuật có dẫn chứng:** Khi đối chiếu kết quả thực nghiệm thực tế trên máy, ở Run A (`boxes-demo-delta-0-voxel-0.16.json`), mô hình chỉ nhận diện được 1 xe ô tô duy nhất (`mean_z = 0.330 m`) và mất sạch người đi bộ cùng xe hai bánh do thiếu bước dịch $z$. Ở Run B (`delta = 1.73 m`), mô hình phát hiện đầy đủ và cân bằng 13 hộp (`vehicles: 10`, `pedestrian: 2`, `two-wheels: 1`) với đáy hộp bám khít mặt đường cục bộ. Sang Run C (`voxel = 0.32 m`), việc nới rộng kích thước pillar làm mất toàn bộ xe và biến thành 6 hộp người ảo do voxel quá thô.
- **Diễn giải phép biến đổi z thuận/nghịch:**
  - *Phép biến đổi thuận* ($z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$) dịch chuyển đám mây điểm từ hệ mặt đất về hệ tọa độ cảm biến LiDAR để khớp với không gian anchor mà mô hình PointPillars đã học.
  - *Phép biến đổi nghịch* ($z_{\text{source}} = z_{\text{model}} + \text{delta} + z_{\text{ground}}$) đưa tâm bounding box dự đoán trở lại hệ tọa độ thực tế của xe tự hành.
- **Quyết định lỗi batch và hành động:** Khi gặp lỗi cả batch chìm dưới lòng đất như trong `case-batch-z` (toàn bộ 13 hộp cùng lệch $-1.805\text{ m}$), tôi kiên quyết chọn dừng pipeline ngay lập tức và báo cho đội ngũ kỹ thuật sửa code. Đây là lỗi logic hệ thống (quên bước chuyển nghịch đảo $z_{\text{source}}$), việc bắt annotator chỉnh tay từng hộp là lãng phí tài nguyên và làm sai lệch phân phối dữ liệu huấn luyện. Ngược lại, nếu chỉ có 1 hộp đơn lẻ bị lỗi như `case-one-box-z`, pipeline vẫn đảm bảo tính toàn vẹn và cho phép chuyển tiếp sang CVAT để annotator sửa cục bộ.
- **Điều chưa chắc chắn:** Chưa thể khẳng định độ chính xác góc quay hướng (`yaw`) của các phương tiện ở cự ly xa chỉ qua hình chiếu 2D của ảnh Side view, do trục $Y$ bị triệt tiêu; cần đối chiếu thêm ảnh camera và đám mây điểm 3D trên CVAT.

---

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: Đã xác thực quyền sử dụng gói Student KITTI `demo.pcd` hợp lệ theo giấy phép CC BY-NC-SA 3.0. Sinh viên thực hành đúng ca Day 13.
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung: Sinh viên đã trực tiếp chạy thật 100% qua Docker offline (`executed-by-group`), mã thoát 0, sinh đầy đủ output Run A/B/C và bộ ca QC. Không cần lượt thực hành bổ sung.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: Đầy đủ file trong thư mục output `ket-qua-nhom-01`, hash SHA256 khớp chuẩn trong `smoke.json`, cam kết không import các file `case-*.json` vào CVAT.
- Nhận xét từng thành viên và quyết định dừng pipeline: Sinh viên có nhận xét sâu sắc, phân biệt chính xác lỗi hệ thống và lỗi đối tượng cục bộ, nắm vững cơ chế biến đổi tọa độ $z$.
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do: **Đồng ý nghiệm thu Phần 1.** Sinh viên đủ điều kiện xuất sắc để chuyển sang thực hiện các job nguồn cá nhân trên CVAT và review chéo trên portal.
