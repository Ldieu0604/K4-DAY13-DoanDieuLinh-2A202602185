# Báo cáo thực hành PointPillars — Day 13

Giữ bản đã điền ngoài Git, trong thư mục nhóm private do LC thu. Đây là kiểm tra formative; không ghi điểm của người khác.

## Nhóm và provenance

- Mã nhóm/phòng: Cá nhân
- Thành viên: xem `TEAMMATES.md` (họ tên/MSSV, vai trò từng lượt).
- Trạng thái: `executed-by-individual`
- Người thực sự chạy; ngày/giờ; hệ máy/architecture: Cá nhân tự chạy; ngày 02/10/2026; Windows Docker Desktop x86_64
- Image tag và image ID; phiên bản repo: day13-pointpillars:lab
- PCD được cấp / frame_id; nơi được phép chạy; fingerprint nếu LC cấp: demo.pcd
- Checkpoint: PointPillars KITTI có sẵn trong image; ghi checkpoint ID/hash nếu LC cấp: Mặc định của gói Student
- Phạm vi: front-window; score threshold: 0.3
- Giả định kênh thứ tư/intensity và nguồn z_ground: RGB=0, z_ground=0.075 (ước lượng từ mặt đất)

## Ba lượt inference thật

A/B/C là ba lượt trên cùng PCD. Runner chạy đủ ba lượt từ một lệnh. Lấy **Số hộp** từ `n_boxes`, **mean_z** từ `mean_z` trong `run-A/B/C/summary.csv`; không tự tính lại hoặc đoán. `mean_z` không phải điểm chất lượng. Mở `side-*.png`, đối chiếu `boxes-*.json` để ghi quan sát. Số hộp không phải đáp án cần khớp nhóm khác.

| Lượt | delta | Pillar XY | Số hộp | mean_z | File JSON/Side/CSV | Quan sát có bằng chứng |
| --- | --- | --- | --- | --- | --- | --- |
| A | 0 | 0.16 | 1 | 0.330 | run-A/boxes-demo-delta-0.0-voxel-0.16.json | Mô hình tìm được rất ít hộp (1 hộp) vì delta chưa điều chỉnh. |
| B | 1.73 | 0.16 | 13 | 1.034 | run-B/boxes-demo-delta-1.73-voxel-0.16.json | Số lượng hộp tăng mạnh (13 hộp), vị trí trục z dịch chuyển phù hợp hơn. |
| C | 1.73 | 0.32 | 6 | 1.091 | run-C/boxes-demo-delta-1.73-voxel-0.32.json | Số lượng hộp giảm còn 6 hộp do kích thước pillar lớn gom nhiều điểm hơn. |

- A/B — chỉ đổi delta: A có 1 hộp; B có 13 hộp. Ảnh/file/vùng `side-*.png` khác ở cao độ trục z của các hộp (B dịch lên 1.73m). Đây là chạy lại model trên input khác, không chỉ dịch hộp cũ; điều em còn chưa chắc là chiều cao chính xác của cảm biến LiDAR thật.
- B/C — chỉ đổi pillar: B có 13 hộp; C có 6 hộp. Ảnh/file/vùng `side-*.png` khác ở số lượng và kích thước hộp. Số lượng/lớp/vị trí thay đổi như sau: Các vật thể nhỏ (người, xe đạp) có thể bị gộp lại hoặc bỏ sót do ô lưới (pillar) quá to. Có đủ bằng chứng để kết luận tốt hơn không? Chưa đủ, vì số lượng ít đi chưa chắc đã chính xác hơn nếu vật nhỏ bị gộp.
- Giới hạn ROI và góc Side ảnh hưởng cách đọc miss/yaw thế nào? Ảnh Side chiếu lên mặt phẳng x-z nên không thấy được sự chồng lấn theo chiều y, do đó không thể xác định chính xác góc yaw (hướng) và rất dễ nhầm lẫn các vật thể đứng che khuất nhau.
- JSON nào còn chưa đủ cơ sở để import? Cả 3 file JSON đều không được import vì đây chỉ là dữ liệu chạy thử (demo KITTI), không phải là frame của xe Robotaxi.

## Ca QC có kiểm soát — không import CVAT

| Ca | Số hộp lệch z / tổng hộp | Lượng lệch | Class/x/y/yaw có đổi? | Dừng batch, kiểm từng hộp hay chưa rõ? | Bằng chứng |
| --- | --- | --- | --- | --- | --- |
| case-correct | 0 / 13 | 0 | Không đổi | Không có lỗi | Đây là bản chuẩn mô phỏng theo B |
| case-batch-z | 13 / 13 | Lệch toàn bộ z_ground + delta | Không đổi | Dừng batch, kiểm pipeline | Toàn bộ 13 hộp đều bị nổi/chìm cùng một khoảng |
| case-one-box-z | 1 / 13 | Lệch z_ground + delta | Không đổi | Kiểm từng hộp | Chỉ có 1 hộp bị trôi z, 12 hộp kia bình thường |

Ghi rõ helper tạo biến đổi có chủ đích từ prediction, không phải kết quả inference riêng hoặc nhãn đúng.

## Nhận xét cá nhân

Tôi thực hiện toàn bộ các vai trò. Nhận thấy khi tăng kích thước Pillar (từ 0.16 lên 0.32), model có xu hướng bỏ sót các vật thể nhỏ do bị gom chung vào một ô lưới lớn. Đối với lỗi z: nếu cả file bị lệch (batch error) thì nguyên nhân từ quá trình transform toạ độ, cần dừng lại báo cáo LC; nếu chỉ 1 hộp bị lệch thì do model nhận diện sai vật thể, cần kiểm tra ở nhiều góc nhìn để sửa hộp đó. Điều chưa chắc chắn là cao độ thực tế của cảm biến (sensor) tại thời điểm thu thập dữ liệu.

## LC ghi nhận riêng

- Quyền dùng PCD/image và đúng ca:
- Có chạy thật / chỉ phân tích; còn cần lượt thực hành bổ sung:
- Output đủ, giữ bản gốc, không đưa ca lỗi vào CVAT:
- Nhận xét từng thành viên và quyết định dừng pipeline:
- Đồng ý chuyển sang chỉnh/QC / cần bổ sung; lý do:
