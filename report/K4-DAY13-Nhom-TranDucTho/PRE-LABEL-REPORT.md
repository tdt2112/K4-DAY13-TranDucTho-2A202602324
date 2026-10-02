# Báo cáo thực hành PointPillars — Day 13

## Nhóm và provenance

- Mã nhóm/phòng: K4-DAY13-TranDucTho-02324
- Thành viên: xem `TEAMMATES.md` (Trần Đức Thọ - MSSV: 02324, phụ trách vận hành và phân tích).
- Trạng thái: `executed-by-group` (trực tiếp chạy runner và kiểm định trên máy với Docker engine).
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Trần Đức Thọ; 2026-10-02; Windows 11 / WSL2 Ubuntu Linux (amd64 / x86_64).
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64` / `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; commit revision: `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: `demo.pcd` / `demo` (17,238 points, chuẩn hóa từ KITTI sample 000008 theo giấy phép CC BY-NC-SA 3.0, SHA256: `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`); chạy offline trong container không mạng.
- Checkpoint: PointPillars KITTI có sẵn trong image `/opt/PointPillars/pretrained/epoch_160.pth` (SHA256: `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`).
- Phạm vi: front-window; score threshold: `0.3`.
- Giả định kênh thứ tư/intensity và nguồn z_ground: Reflectance thực tế của KITTI được lược bỏ trong bản chuyển đổi, thay bằng RGB=0 placeholder; adapter sử dụng kênh hằng số; $z_{\text{ground}} = 0.075\text{ m}$ được ước lượng tự động từ điểm mặt đất của PCD nguồn.

## Ba lượt inference thật

Lấy số liệu trực tiếp từ `summary.csv`, `boxes-*.json` và đối chiếu trên ảnh `side-*.png` của thư mục `ket-qua-nhom-01`:

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| :---: | :---: | :---: | :---: | :---: | :--- | :--- |
| **A** | 0 m | 0.16 m | 1 | 0.330 m | `run-A/summary.csv`, `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png` | Chỉ phát hiện 1 hộp duy nhất (`vehicles: 1`) tại vị trí gần cảm biến ($x \approx 6.4\text{ m}, y \approx 1.5\text{ m}$), toàn bộ các phương tiện khác trong tầm nhìn bị bỏ sót. |
| **B** | 1.73 m | 0.16 m | 13 | 1.034 m | `run-B/summary.csv`, `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png` | Phát hiện 13 hộp (`vehicles: 10`, `pedestrian: 2`, `two-wheels: 1`) phân bố trải dài dọc trục x từ 4 m đến trên 30 m, bám sát các cụm điểm phương tiện. |
| **C** | 1.73 m | 0.32 m | 6 | 1.091 m | `run-C/summary.csv`, `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png` | Số hộp giảm xuống 6 hộp và 100% bị gán nhầm thành `pedestrian: 6`, không còn nhận diện được lớp `vehicles`. |

- **A/B — chỉ đổi delta:** A có 1 hộp; B có 13 hộp. Ảnh `side-demo-delta-1.73-voxel-0.16.png` của B xuất hiện hàng loạt cụm hộp bao trọn các đối tượng ở dải $x \in [10, 30]\text{ m}$ mà ở ảnh A hoàn toàn trống rỗng. Đây là việc chạy lại mô hình trên phân bố voxel đầu vào đã được bù chiều cao đặt cảm biến ($z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$), giúp mạng neural nhận dạng đúng cấu trúc hình học đã học; hoàn toàn khác với việc tịnh tiến hộp sau inference. Điều em còn chưa chắc là giá trị delta = 1.73 m có tối ưu cho địa hình dốc hay mấp mô lớn hay không.
- **B/C — chỉ đổi pillar:** B có 13 hộp; C có 6 hộp. Khi tăng cạnh pillar từ 0.16 m lên 0.32 m (diện tích cột tăng gấp 4 lần), độ phân giải không gian của voxel bị suy giảm nghiêm trọng. Model bị mất chi tiết hình học của xe và dự đoán sai toàn bộ thành `pedestrian`. Không có đủ bằng chứng để kết luận C tốt hơn (thực tế C kém hơn rõ rệt vì checkpoint pretrained được tối ưu hóa cho voxel 0.16 m).
- **Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào?** Checkpoint chỉ inference cửa sổ phía trước (front-window), nên các đối tượng ngoài ROI không được coi là lỗi bỏ sót (miss). Ảnh Side là hình chiếu trực giao $x-z$ gộp chung toàn scene, các vật thể ở các tọa độ y khác nhau bị chồng lên nhau, do đó ảnh Side chỉ giúp phát hiện nhanh lỗi cao độ $z$ chứ không thể dùng để kết luận góc xoay yaw hay độ khớp ngang của từng đối tượng.
- **JSON nào còn chưa đủ cơ sở để import? Cần kiểm gì tiếp?** Cả 3 file JSON đều không đủ cơ sở để coi là nhãn ground truth: Run A thiếu đối tượng nghiêm trọng, Run C sai class hàng loạt, Run B tuy số lượng tốt nhưng vẫn là output thô của AI (có thể sai hướng 180°, đáy chưa bám mặt đường hoặc sai kích thước khi bị che khuất). Cần kiểm tra kỹ từng hộp trên 4 góc nhìn và đối chiếu camera. Tuyệt đối không import các file này vào bài Robotaxi vì khác tập dữ liệu.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| :--- | :---: | :---: | :---: | :--- | :--- |
| **`case-correct`** | 0 / 13 | 0 m | Không đổi | **Chấp nhận làm baseline đối chiếu** | Tọa độ và class giữ nguyên bản gốc từ Run B (`boxes-demo-delta-1.73-voxel-0.16.json`). |
| **`case-batch-z`** | 13 / 13 | -1.805 m ($\Delta + z_{\text{ground}}$) | Không đổi | 🛑 **Dừng batch, báo LC kiểm tra pipeline** | Toàn bộ 13/13 hộp đồng loạt chìm xuống lòng đất đúng -1.805 m trong khi x, y, class không đổi. Đây là lỗi quên phép biến đổi ngược của cả pipeline; không được sửa tay từng hộp. |
| **`case-one-box-z`** | 1 / 13 | -1.805 m | Không đổi | 🔍 **Kiểm tra và sửa từng hộp (Object-level)** | Duy nhất hộp đầu tiên bị chìm, 12 hộp còn lại nằm đúng mặt đất. Đây là lỗi đối tượng đơn lẻ, kiểm tra bằng nhiều góc nhìn để nắn chỉnh. |

*Lưu ý: Helper `pipeline-qc-cases.py` tạo biến đổi có chủ đích từ prediction Run B để phục vụ huấn luyện nhận diện lỗi pipeline, không phải kết quả detector riêng biệt hay nhãn ground truth.*

## Nhận xét cá nhân

- **Thành viên: Trần Đức Thọ (MSSV: 02324)**
  - **Vai trò:** Trực tiếp vận hành lệnh runner Docker, kiểm tra môi trường native Linux amd64, đọc và trích xuất dữ liệu từ `summary.csv`, JSON và ảnh Side view; viết báo cáo phân tích.
  - **Quan sát A/B/C:** Khi đối chiếu file `run-A/boxes-demo-delta-0-voxel-0.16.json` và `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, nhận thấy việc đổi delta làm số hộp tăng từ 1 lên 13. Điều này khẳng định phép trừ delta trước inference là điều kiện tiên quyết để phân bố điểm của PointPillars khớp với không gian chuẩn của checkpoint pretrained KITTI.
  - **Diễn giải phép biến đổi z:**
    $$z_{\text{model}} = z_{\text{source}} - z_{\text{ground}} - \text{delta}$$
    $$z_{\text{source}} = z_{\text{model}} + z_{\text{ground}} + \text{delta}$$
    Phép đổi thuận chuẩn hóa điểm về hệ tọa độ của mô hình (gốc z tại đáy cảm biến/mặt đường). Phép đổi ngược đưa tâm hộp dự đoán trở lại hệ tọa độ cảm biến LiDAR nguồn để khớp với đám mây điểm gốc.
  - **Quyết định lỗi batch:** Khi gặp hiện tượng như `case-batch-z` (tất cả các hộp đều lệch cùng một hằng số chiều cao), em quyết định dừng thao tác sửa nhãn thủ công và báo ngay cho kỹ sư pipeline/LC để sửa code biến đổi ngược, tránh lãng phí thời gian sửa từng hộp vô ích.
  - **Điều chưa chắc:** Việc ước lượng $z_{\text{ground}}$ dựa trên thuật toán đơn giản có thể bị sai lệch tại các đoạn đường có độ dốc lớn hoặc mấp mô gồ ghề.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
