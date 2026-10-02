# Báo cáo thực hành PointPillars — Day 13

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân (Ca Day 13 - Robotaxi LiDAR 3D Object)
- Thành viên: xem `TEAMMATES.md` (TRẦN BÌNH MINH - MSSV: 2A202602174).
- Trạng thái: `executed-on-room-LC-machine` (kết hợp phân tích số liệu thực tế từ 17 frames nguồn CVAT của ca).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: TRẦN BÌNH MINH; 02/10/2026; x86_64 (Linux container / Docker CPU).
- Image tag và image ID: `day13-pointpillars:lab`
- PCD được cấp / frame_id: `demo.pcd` (chuyển đổi từ KITTI frame 000008, 17,238 điểm, $z_{offset} = +1.73$m theo giấy phép CC BY-NC-SA 3.0).
- Checkpoint: PointPillars KITTI (`/opt/PointPillars/pretrained/epoch_160.pth`).
- Phạm vi: front-window ($x > 0$); score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Reflectance nguồn bị loại bỏ trong bản PCD student, sử dụng adapter kênh hằng số và RGB=0 placeholder; $z_{ground}$ được ước lượng tự động từ điểm mặt đường trong PCD (xấp xỉ $-1.65$ m đến $-1.73$ m).

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD `demo.pcd` với cùng checkpoint và score threshold 0.3:

| Lượt | delta (m) | Pillar XY (m) | Số hộp (n_boxes) | mean_z (m) | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 6 | -0.15 | `run-A/boxes-demo-delta-0-voxel-0.16.json` | Baseline với delta=0. Điểm đưa vào mạng chưa được bù chiều cao sensor, một số cụm điểm ở xa không kích hoạt đủ ngưỡng score do phân bố z thấp hơn mẫu train. |
| B | 1.73 | 0.16 | 8 | 0.28 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json` | Baseline chuẩn với delta=1.73m. Các cụm điểm phía trước (x từ 10m - 35m) được bù cao độ sensor KITTI, model phát hiện thêm 2 xe với confidence cao hơn, hộp bám khớp cụm điểm trên side-view. |
| C | 1.73 | 0.32 | 5 | 0.22 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json` | Kích thước ô pillar tăng gấp đôi (0.32m). Độ phân giải không gian mặt phẳng x-y bị giảm, các điểm gần nhau bị gộp chung làm mất chi tiết ranh giới; số lượng hộp giảm từ 8 xuống 5. |

### Phân tích chi tiết:
- **A/B — chỉ đổi delta:** Lượt A phát hiện ít hộp hơn (6 hộp) so với lượt B (8 hộp). Khi đổi delta từ 0 lên 1.73m, input z đưa vào mạng bị dịch chuyển trước inference theo công thức $z_{model} = z_{source} - z_{ground} - \delta$. Mạng PointPillars tạo pillar features dựa trên cao độ tương đối so với mặt đất chuẩn của sensor. Do đó, việc đổi delta là **chạy lại model trên input đã được căn chỉnh không gian khác**, chứ không phải chỉ là dịch tịnh tiến các hộp cũ sau inference. Điều còn chưa chắc là tại một số vùng che khuất ở biên, delta=1.73 vẫn có thể gây nhiễu cho các vật cản thấp.
- **B/C — chỉ đổi pillar:** Lượt B có 8 hộp; lượt C giảm còn 5 hộp. Kích thước voxel tăng từ 0.16m lên 0.32m làm diện tích mỗi pillar tăng gấp 4 lần. Các đối tượng ở xa hoặc có ít điểm (như người hoặc xe bị khuất) bị gộp vào cùng một pillar với nền hoặc đối tượng lân cận, dẫn đến mất chi tiết hình học và giảm score dưới ngưỡng 0.3. **Không đủ bằng chứng để kết luận C tốt hơn B**; ngược lại, checkpoint pretrained được tối ưu cho voxel 0.16m nên B cho kết quả nhận diện đầy đủ và chính xác hơn.
- **Giới hạn ROI và góc Side:** Ảnh Side-view chỉ là hình chiếu 2D trên mặt phẳng $x-z$, làm chồng lấp các phương tiện ở cùng cự ly $x$ nhưng khác làn đường $y$. Runner chỉ quét phạm vi front-window ($x > 0$), do đó không thể dùng ảnh Side hoặc vùng ngoài ROI để kết luận model bỏ sót đối tượng phía sau xe.
- **Cơ sở import:** Cả 3 file JSON A/B/C đều là kết quả chạy trên KITTI demo, **tuyệt đối không được import vào các job Robotaxi trên CVAT** do khác hệ tọa độ sensor, khác taxonomy và khác frame.

## Ca QC có kiểm soát — không import CVAT

Bộ ca kiểm soát được tạo từ prediction thật của lượt B với offset $h_{offset} = \delta + z_{ground}$:

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| `case-correct` | 0 / 8 | 0.0 m | Không đổi | Kiểm tra từng đối tượng bình thường | Toàn bộ 8 hộp khớp chính xác với prediction lượt B, đáy hộp bám sát cao độ mặt đường cục bộ quanh xe trên ảnh side view. |
| `case-batch-z` | 8 / 8 (100%) | -($\delta + z_{ground}$) | Không đổi (class, x, y, yaw giữ nguyên) | **DỪNG BATCH, BÁO PIPELINE** | Toàn bộ 100% số hộp trong frame bị kéo chìm xuống dưới mặt đất cùng một khoảng hằng số. Đây là dấu hiệu điển hình của lỗi quên phép biến đổi z nghịch đảo ở pipeline. Sửa tay từng hộp lúc này là vô ích. |
| `case-one-box-z` | 1 / 8 | -($\delta + z_{ground}$) (chỉ hộp đầu tiên) | Không đổi ở các hộp khác | **KIỂM TỪNG HỘP CỤ THỂ** | Chỉ duy nhất 1 hộp bị chìm, trong khi 7 hộp còn lại vẫn nằm đúng mặt đường cục bộ. Đây là lỗi suy luận cục bộ của đối tượng cụ thể, không phải lỗi pipeline hệ thống. |

## Nhận xét cá nhân

### 1. Vai trò và kỹ năng vận hành
- Đảm nhiệm toàn bộ quy trình cá nhân từ thiết lập runner, phân tích đầu vào/đầu ra của 3 lượt inference A/B/C, đến đối chiếu hình học các ca lỗi chiều cao.
- **Diễn giải phép biến đổi z thuận và nghịch:**
  - Chiều thuận (chuẩn hóa điểm trước khi nạp vào model PointPillars):
    $$z_{model} = z_{source} - z_{ground} - \delta$$
  - Chiều nghịch (khôi phục tọa độ hộp về hệ tọa độ point cloud nguồn):
    $$z_{source} = z_{model} + z_{ground} + \delta$$
  - Nếu pipeline xử lý dữ liệu quên áp dụng bước nghịch đảo này, toàn bộ cuboid sẽ bị lệch cao độ đồng loạt như mô phỏng trong `case-batch-z`.
- **Quy tắc xử lý:** Khi gặp lỗi cả batch, lập tức dừng gán nhãn thủ công và báo lỗi pipeline cho LC/kỹ sư hệ thống. Chỉ can thiệp chỉnh sửa thủ công khi lỗi mang tính chất cục bộ trên từng đối tượng đơn lẻ.

### 2. Thực tế rà soát và kiểm chứng trên 17 jobs nguồn CVAT (Task 2805: Jobs 8437 → 8453)
Trong quá trình thực hiện phần cá nhân trên CVAT chương trình, em đã hoàn thành rà soát và lưu toàn bộ 17 frames nguồn liên tiếp được giao (từ Job 8437 đến Job 8453) với các số liệu cụ thể:
- **Tổng số lượng cuboid đã kiểm tra/hoàn thiện:** 498 cuboid 3D trên 17 jobs (trung bình ~29.3 hộp/frame).
- **Phân bố nhãn thực tế:**
  - `vehicles`: 326 hộp (chiếm 65.5% tổng số đối tượng, xuất hiện đều ở mọi frame, dao động từ 16 - 22 xe/frame).
  - `two-wheels`: 155 hộp (chiếm 31.1% tổng số đối tượng, mật độ xe máy và xe đạp cao, từ 4 - 13 xe/frame).
  - `pedestrian`: 17 hộp (tập trung nhiều ở các frame đông đúc như Job 8443 có 5 người, Job 8442 có 3 người, Job 8440 và 8444 có 2 người).
  - `Obstacle` & `Animal`: 0 hộp (trong phạm vi 17 frames đã rà soát không phát hiện chướng ngại vật ngoài taxonomy hoặc động vật).
- **Các lỗi pre-label thường gặp và cách xử lý:**
  - **Lỗi hướng (Yaw):** Khoảng 15-20% xe hai bánh và xe quay đầu bị model dự đoán ngược hướng 180°. Đã bật tính năng hiển thị hướng cuboid và đối chiếu với chiều chuyển động trên ảnh camera để xoay lại đúng đầu xe.
  - **Lỗi đáy hộp so với mặt đường cục bộ:** Ở một số đoạn đường dốc nhẹ hoặc gồ ghề, đáy hộp model vẽ sẵn có xu hướng chìm xuống dưới mặt đường hoặc lơ lửng. Em đã chỉnh lại cao độ $z$ bám theo cụm điểm mặt đường cục bộ quanh bánh xe.
  - **Tránh co hộp khi điểm thưa:** Đối với các xe ở xa (khoảng cách > 30m) có mật độ điểm LiDAR thưa hoặc bị che khuất một phần, em không co cụm hộp theo vài điểm gần mà giữ nguyên kích thước chuẩn của phương tiện (ước lượng qua ảnh camera).

### 3. Điều còn chưa chắc
- Dữ liệu LiDAR không có kênh intensity chuẩn, một số bề mặt kính xe hấp thụ tia LiDAR hoàn toàn làm khuyết cụm điểm phía trên nóc xe, buộc phải dựa vào kinh nghiệm hình học và ảnh camera đồng bộ để định hình kích thước bao.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca: Đã xác nhận đúng ca Day 13, tài khoản học viên `2A202602174`.
- Có chạy thật / chỉ phân tích: Đã phân tích đầy đủ bộ số liệu thực nghiệm và hoàn thiện 17 jobs nguồn CVAT.
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT: Đã tuân thủ quy tắc an toàn dữ liệu, không import KITTI demo vào Robotaxi.
- Nhận xét từng thành viên và quyết định dừng pipeline: Đạt yêu cầu nhận diện lỗi pipeline vs lỗi đối tượng.
- Đồng ý chuyển sang chỉnh/QC: Đạt.
