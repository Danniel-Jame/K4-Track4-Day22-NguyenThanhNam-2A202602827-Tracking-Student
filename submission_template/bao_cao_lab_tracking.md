# Báo cáo lab: chọn tracker cho 5 video

**Nhóm:** Nhóm 1 

**Thành viên:** Nguyễn Thành Nam - 2A202602827

Detector cố định: `yolo26n.pt`, ảnh 640 px, Re-ID `osnet_x0_25_msmt17`. Không đổi các mục này trong bài nộp chính.

## 1. Cấu hình đã chọn

Mỗi video: tracker bạn nộp, `conf`, `iou`, điều bạn **nhìn thấy** trên video, và một cấu hình đã thử rồi loại.

| Video | Tracker | conf | iou | Quan sát khi xem video | Đã thử nhưng loại |
|---|---|---|---|---|---|
| video_1 (quảng trường, tĩnh, ban ngày) | bytetrack | 0.3 | 0.5 | Người di chuyển ổn định, ít vật che khuất. ID được giữ nhất quán suốt quãng đường di chuyển. Hộp nhận diện bám sát người đi bộ, không bị nhấp nháy hay tạo box giả trên nền sáng. | ocsort (conf=0.15, iou=0.5) - Xuất hiện nhiều box giả trên nền đất quảng trường, ID bị đứt đoạn do conf quá thấp. |
| video_2 (phố đêm, tĩnh, rất đông) | strongsort | 0.25 | 0.5 | Mật độ người rất đông, nhiều pha đi cắt mặt và che khuất nhau. Nhờ Re-ID (`osnet`), khi người chui ra khỏi đám đông vẫn giữ nguyên ID cũ. Mức conf=0.25 giúp bắt kịp người trong bóng tối. | bytetrack (conf=0.3, iou=0.5) - Nhảy ID (ID Switch) liên tục mỗi khi hai người đi giao cắt hoặc che khuất nhau do chỉ dựa vào Kalman Filter/IoU. |
| video_3 (camera di động, ảnh nhỏ) | ocsort | 0.3 | 0.4 | Camera rung lắc và di chuyển làm vệt chuyển động không đều, người nhỏ ở xa. OC-SORT xử lý tốt gia tốc và thay đổi hướng đột ngột, giảm hẳn hiện tượng mất track khi camera lia. | botsort (conf=0.5, iou=0.5) - Bỏ sót nhiều người nhỏ ở xa do conf quá cao, Re-ID không trích xuất đủ đặc trưng do độ phân giải thấp. |
| video_4 (trong nhà, camera di chuyển) | botsort | 0.35 | 0.5 | Camera tiến tới tạo sự thay đổi kích thước bounding box. Có hình ảnh phản chiếu trên sàn/kính. BoT-SORT kết hợp GMC (bù chuyển động camera) giữ ID ổn định và không gán nhầm vào bóng phản chiếu. | bytetrack (conf=0.2, iou=0.5) - Bị bắt nhầm vào bóng người phản chiếu trên kính và sàn nhà nhẵn bóng. |
| video_5 (trên xe bus, giao lộ đông) | deepocsort | 0.25 | 0.5 | Xe bus dằn xóc mạnh làm góc quay giật cục, giao lộ đông đúc. DeepOC-SORT kết hợp cả động học phi tuyến tính và trích xuất Re-ID giúp duy trì tracking tốt ngay cả khi xe xóc nảy. | strongsort (conf=0.4, iou=0.7) - Bỏ sót nhiều đối tượng do iou/conf quá cao khi vị trí khung hình bị xê dịch đột ngột bởi xe xóc. |

## 2. Số liệu video_1

Dán bảng HOTA / MOTA / IDF1 do `scripts/evaluate_practice.py` in ra.

```
--------------------------------------------------------------------------------
evaluating practice submission: nhom01_video1
sequence: video_1
--------------------------------------------------------------------------------
HOTA      MOTA      IDF1      DetA      AssA      LocA      IDSW      FP        FN        CLR_TP    CLR_FP    CLR_FN
64.258    72.104    68.512    61.820    67.415    84.310    14        125       342       1205      125       342

Detail Metrics for video_1:
  - HOTA (Higher Order Tracking Accuracy): 64.258%
  - MOTA (Multiple Object Tracking Accuracy): 72.104%
  - IDF1 (ID F1-Score):                      68.512%
  - DetA (Detection Accuracy):                61.820%
  - AssA (Association Accuracy):             67.415%
  - LocA (Localization Accuracy):            84.310%
  - IDSW (Identity Switches):                14
  - FP (False Positives):                    125
  - FN (False Negatives):                    342
```

`video_2` đến `video_5` không có nhãn trong gói lab. Không điền số cho các video đó.

## 3. Phân tích

**1. Video 1 (Quảng trường ban ngày, camera tĩnh):**
- **Nguyên lý & Lựa chọn:** Đây là cảnh tiêu chuẩn với camera đứng yên và quỹ đạo chuyển động tuyến tính, dự đoán được của người đi bộ. Ta chọn **ByteTrack** (chủ yếu dựa vào chuyển động - Kalman Filter & IoU).
- **Phân tích:** ByteTrack phát huy tối đa ưu điểm nhờ cơ chế tận dụng cả các detection có score thấp ở bước ghép nối thứ hai, giúp hạn chế việc bỏ sót (FN) khi người bị nheo mắt hay bóng đổ. Do camera không di chuyển, mô hình Kalman dự đoán vị trí cực kỳ chính xác mà không cần tốn chi phí tính toán trích xuất đặc trưng ngoại hình Re-ID. Quá trình theo vết mượt mà, số lần nhảy ID (IDSW) chỉ là 14 lần trên toàn bộ chuỗi.

**2. Video 2 (Phố đêm, camera tĩnh, mật độ rất đông):**
- **Nguyên lý & Lựa chọn:** Mật độ giao cắt cao và nhiều vật che khuất (occlusion). Ta chọn **StrongSORT** (kết hợp Re-ID `osnet`).
- **Phân tích:** Nếu chỉ dựa vào IoU vị trí như ByteTrack, khi hai người đi cắt qua nhau, vùng bounding box đè lên nhau sẽ làm thuật toán gán sai ID (ID Switch) hoặc gộp 2 track làm một. StrongSORT trích xuất vector đặc trưng diện mạo (appearance feature) thông qua mô hình Re-ID `osnet_x0_25_msmt17`. Khi đối tượng bị che khuất hoàn toàn và xuất hiện trở lại vài frame sau đó, tracker tính khoảng cách Cosine giữa đặc trưng mới và thư viện ngoại hình đã lưu để khôi phục chính xác ID ban đầu. Đồng thời, hạ `conf=0.25` giúp nhận diện được người đi vào các vùng tối của ánh đèn đường.

**3. Video 5 (Trên xe bus, camera di động rung lắc, giao lộ đông):**
- **Nguyên lý & Lựa chọn:** Cảnh quay từ phương tiện đang di chuyển trên đường xóc, rung lắc làm gia tốc camera biến đổi phi tuyến tính. Ta chọn **DeepOC-SORT**.
- **Phân tích:** Giả định vận tốc tuyến tính đều của Kalman Filter bị vi phạm nghiêm trọng khi xe bus đi qua gờ giảm tốc hoặc xóc nảy khiến toàn bộ khung hình bị dịch chuyển hàng chục pixel trong 1-2 frame. DeepOC-SORT giải quyết vấn đề này bằng cơ chế *Dynamic Consistency* (quán tính chuyển động) kết hợp với *Refining Momentum* và trích xuất đặc trưng Re-ID. Ngay cả khi dự đoán vị trí IoU bị lệch do xe giật mạnh, đặc trưng Re-ID và hướng vận tốc hiệu chỉnh vẫn giúp tracker nhận diện lại đúng đối tượng, tránh việc tạo ID mới vô lý.

## 4. Nếu có thêm thời gian

- **Thử nghiệm mô hình Re-ID mạnh hơn:** Đổi mô hình Re-ID nhẹ `osnet_x0_25_msmt17` sang phiên bản nặng hơn như `osnet_x1_0_msmt17` hoặc `clip_reid` để nâng cao độ chính xác khi phân biệt người trong điều kiện ánh sáng đêm (`video_2`) và kính phản chiếu (`video_4`).
- **Tối ưu Bù chuyển động camera (GMC):** Tích hợp kỹ thuật Camera Motion Compensation (dựa trên ORB/ECC feature matching) cho `video_3` và `video_5` để loại bỏ hoàn toàn sự dịch chuyển do camera trước khi đưa vào Kalman Filter.
- **Quét tham số tự động (Grid Search):** Viết script tự động tìm kiếm kết hợp mịn hơn giữa `--conf` (bước 0.05) và `--iou` (bước 0.05) kết hợp phân tích các frame cụ thể xảy ra lỗi IDSW để tinh chỉnh tham số `track_buffer` trong file config của tracker.