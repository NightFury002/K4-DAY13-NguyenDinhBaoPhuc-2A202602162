# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục cá nhân/private do LC thu. Đây là kiểm tra formative.

## Người thực hiện và provenance

- Mã ca/phòng: H210
- Người thực hiện: xem `TEAMMATES.md` (một người, tự đảm nhiệm A/B/C và QC).
- Trạng thái: `executed-by-solo`; `smoke.json` báo `passed`.
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: [TÊN/MSSV BỔ SUNG]; 2026-10-02 14:34–14:37 UTC; Linux container / amd64.
- Image tag và image ID; phiên bản repo: `day13-pointpillars:lc-20261001-amd64`; `sha256:e03983bd922ec29890bf547db8de408402efd82583680b62e671c20da2fd2c82`; repo revision trong bundle `0831856d921609312d42c7582c366e5a311bb7b1`.
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: Student KITTI `input/demo.pcd`, frame `demo`, SHA256 `3b5ea3da13e2b19149cab6a8d521c2ca55f2df93f026b5a3f8c273ce70645d60`; chạy local Docker Desktop.
- Checkpoint: PointPillars KITTI có sẵn trong image; `/opt/PointPillars/pretrained/epoch_160.pth`, SHA256 `482dfcf63b932cc5ccf012b4bbdad52aa51aa33becf87d0a39d61c39b377b5b1`.
- Phạm vi: front-window; score threshold: 0.3 theo runner chuẩn.
- Giả định kênh thứ tư/intensity và nguồn z_ground: PCD Student dùng trường `rgb` hằng/placeholder, không phải intensity LiDAR thật; `z_ground` do pipeline ước lượng từ PCD.

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD do một người thực hiện. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp người khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | `run-A/boxes-demo-delta-0-voxel-0.16.json`, `run-A/side-demo-delta-0-voxel-0.16.png`, `run-A/summary.csv` | 1 `vehicles`, x≈13.15, y≈−0.45, score≈0.322 |
| B | 1.73 | 0.16 | 13 | 1.034 | `run-B/boxes-demo-delta-1.73-voxel-0.16.json`, `run-B/side-demo-delta-1.73-voxel-0.16.png`, `run-B/summary.csv` | 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; output trải khoảng x≈3–41 |
| C | 1.73 | 0.32 | 6 | 1.091 | `run-C/boxes-demo-delta-1.73-voxel-0.32.json`, `run-C/side-demo-delta-1.73-voxel-0.32.png`, `run-C/summary.csv` | 6 `pedestrian`; không có `vehicles`/`two-wheels` trong output C |

- A/B — A có 1 hộp và mean_z 0.330; B có 13 hộp và mean_z 1.034. JSON cho thấy A chỉ có một `vehicles`, còn B có 10 `vehicles`, 2 `pedestrian`, 1 `two-wheels`; số hộp, class, score và vị trí đều thay đổi. Đây là chạy lại model trên input đã đổi z, không chỉ dịch hộp cũ; chưa đủ bằng chứng để kết luận B đúng hơn.
- B/C — B có 13 hộp và mean_z 1.034; C có 6 hộp và mean_z 1.091. Khi chỉ đổi pillar từ 0.16 lên 0.32, C cho 6 `pedestrian` và không có `vehicles`/`two-wheels`, cho thấy output thay đổi mạnh theo biểu diễn đầu vào. Không kết luận C tốt hơn chỉ vì mean_z cao hơn hoặc B tốt hơn vì có nhiều hộp hơn.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Front-window loại vùng ngoài ROI khỏi kết luận miss; Side là phép chiếu x-z nên có thể chồng các vật thể khác y. Yaw cần kiểm thêm Top/Front và camera, không kết luận từ Side một mình.
- JSON nào còn chưa đủ cơ sở để import? Hiện chưa có JSON inference. Khi có output, prediction KITTI này chỉ dùng để học; không import vào CVAT Robotaxi. Cần kiểm `frame_id`, dataset, delta, voxel size, z_ground, class, đủ 7 trường hình học, ảnh nhiều góc và quyền import đúng job.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0/13 | 0 | Không đổi | Không có lỗi z trong case đối chứng | `qc-cases/case-correct.json`, `qc-cases/side-correct.png` |
| case-batch-z | 13/13 | 1.805 m (`delta + z_ground = 1.73 + 0.075`) | Class/x/y/yaw giữ nguyên; z bị hạ đồng loạt | Dừng batch, kiểm transform/pipeline | `qc-cases/case-batch-z.json`, `qc-cases/side-batch-z.png` |
| case-one-box-z | 1/13 | 1.805 m trên hộp đầu tiên | Chỉ z của hộp đầu tiên đổi; class/x/y/yaw giữ nguyên | Kiểm từng hộp qua nhiều view; chưa kết luận pipeline | `qc-cases/case-one-box-z.json`, `qc-cases/side-one-box-z.png` |

Ghi rõ: helper tạo biến đổi có chủ đích từ prediction B, không phải kết quả inference riêng và không phải nhãn đúng. Cần có prediction B hợp lệ (ít nhất 2 hộp) trước khi tạo được ba file case.

## Nhận xét của người thực hiện

Người thực hiện: [TÊN — MSSV]. Em tự vận hành runner, đọc JSON/CSV, xem Side và thực hiện QC. A có 1 hộp, B có 13 hộp và C có 6 hộp; bằng chứng nằm trong `summary.csv`, JSON và ảnh Side của từng lượt. Pipeline dùng `z_model = z_source - z_ground - delta` và hoàn nguyên bằng `z_source = z_model + z_ground + delta`; với `z_ground=0.075`, case batch/one-box mô phỏng lệch 1.805 m. Quyết định QC: batch-z phải dừng để kiểm transform; one-box-z kiểm từng hộp. Điều chưa chắc: số hộp/class thay đổi không chứng minh cấu hình nào đúng hơn; cần đối chiếu hình học và reference được duyệt.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét người thực hiện và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
