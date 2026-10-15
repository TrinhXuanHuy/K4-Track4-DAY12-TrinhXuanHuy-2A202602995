# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | Trịnh Xuân Huy |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | 2A202602995 |
| GitHub | TrinhXuanHuy |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 3341 / 9005 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 3 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.251 | 20.4 % | BLOCK |
| IDM | 1.503 | 0.1 % | (baseline) |
| EgoMLP | 0.761 .. 0.786 | 2.6 .. 31.5 % | BLOCK |
| PercMLP | 0.622 .. 0.631 | 0.5 .. 6.0 % | BLOCK |
| PercMLP+ (sau khi sửa) | 0.820 | 0.3 .. 0.3 % | RELEASE |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Trong Bước 0, kiểm toán dữ liệu tại evidence/data_audit.json phát hiện ba vấn đề nghiêm trọng. Thứ nhất, cột gia tốc thô của xe dẫn Acc_LV có hệ số tương quan với đạo hàm vận tốc chỉ đạt 0.0454 (gần bằng 0), trong khi xe sau đạt 0.9291, chứng tỏ cảm biến ghi nhận Acc_LV bị lỗi nghiêm trọng hoặc nhiễu không sử dụng được; hệ thống bắt buộc phải thay thế bằng gia tốc làm mượt a_lead tính từ đạo hàm của chuỗi vận tốc làm mượt v_lead_s. Thứ hai, có 383 quỹ đạo trùng lặp tuyệt đối (exact duplicates) bị loại bỏ trong tổng số 9993 quỹ đạo thô ban đầu, giữ lại 9610 quỹ đạo sạch (trajectories_clean). Thứ ba, việc chia tập train/test bắt buộc phải thực hiện theo đơn vị toàn bộ quỹ đạo (trajectory) với tỷ lệ 70/30 thay vì chia ngẫu nhiên theo khung hình (frame), bởi vì các khung hình trong cùng quỹ đạo cách nhau 0.1 s có tính phụ thuộc thời gian rất cao; nếu chia theo khung hình sẽ gây rò rỉ dữ liệu (data leakage) giữa tập học và tập kiểm thử.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Theo evidence/replay.json, bản refactor chuyển sang float32 và đổi thứ tự cộng khác bit ở toàn bộ 600/600 đoạn (100% not_bit_identical) do tính chất không kết hợp của số thực ((0.1+0.2)+0.3 = 0.6000000000000001 khác 0.1+(0.2+0.3) = 0.6 với độ lệch tương đối 6.94e-06), nhưng có đúng 0/600 đoạn vượt dung sai cho phép (atol=1e-3, rtol=1e-3) với độ lệch tuyệt đối lớn nhất chỉ 1.22e-04 m/s^2. Phiên bản thay đổi logic thêm phanh khẩn cấp AEB làm lệch đúng 10/600 đoạn (gồm các đoạn 69, 168, 231, 300, 303...). Khi mở đoạn 69 ra phân tích, lệnh điều khiển chỉ sai lệch cục bộ tại những khung hình nguy hiểm khi TTC giảm xuống dưới ngưỡng 1.5 s, khiến AEB can thiệp phanh khẩn cấp với độ lệch gia tốc cực đại đạt 1.98 m/s^2. Chế độ phát theo thời gian thực (realtime) làm lộ ra 583 khung trễ (late) và 574 khung rớt (dropped) trên 59733 khung hình, làm thay đổi 371/600 đoạn; trong khi chế độ phát nhanh nhất (fastest) che giấu hoàn toàn các vi phạm thời hạn tính toán này với 0 late và 0 dropped.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Theo evidence/scenarios.json, lưới kịch bản cắt đầu được thiết lập với vận tốc cruise 12.0 m/s, khoảng cách từ 2.76 m đến 59.31 m, tốc độ tiếp cận từ 0.0 đến 12.0 m/s, gồm tổng cộng 2700 ô (grid_cells). Ở độ trễ phản xạ t_react = 0.0 s, có 2.22% số ô (unavoidable_area) là bất khả thi về mặt vật lý mà không thuật toán nào tránh được; khi độ trễ tăng lên 0.6 s, vùng bất khả kháng này mở rộng lên 7.44% số ô. Thuật toán IDM bám sát hoàn hảo giới hạn vật lý này: diện tích va chạm của IDM tăng tương ứng từ 2.22% lên 7.44%, nghĩa là diện tích va chạm do lỗi thuật toán (attributable_area) bằng đúng 0.00% ở cả hai độ trễ. Tập dữ liệu log hoàn toàn không bao phủ được các tình huống phanh khẩn cấp: mức phanh gắt nhất của tài xế người trong log (phân vị 99.9%) chỉ đạt 3.62 m/s^2 (human_brake_limit_p99.9), trong khi cơ cấu phanh tối đa đạt 9.0 m/s^2. Có tới 8.56% diện tích lưới kịch bản đòi hỏi lực phanh vượt quá 3.62 m/s^2, nằm ngoài vùng dữ liệu mà mô hình học máy từng được quan sát trong log.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

Bảng so sánh từ evidence/closedloop.json:
- PercMLP: L2 vòng hở 0.626 m (Hạng 1) | Va chạm vòng kín 1.7% [0.5% .. 6.0%] (Hạng 2)
- EgoMLP: L2 vòng hở 0.771 m (Hạng 2) | Va chạm vòng kín 16.2% [2.6% .. 31.5%] (Hạng 3)
- const-vel: L2 vòng hở 1.251 m (Hạng 3) | Va chạm vòng kín 20.4% (Hạng 4)
- IDM: L2 vòng hở 1.503 m (Hạng 4) | Va chạm vòng kín 0.1% (Hạng 1)
Có 3 cặp bị đảo ngược thứ hạng: ('const-vel', 'IDM'), ('IDM', 'EgoMLP'), ('IDM', 'PercMLP'). Thuật toán IDM có sai số L2 vòng hở tệ nhất (1.503 m) vì nó không cố khớp các biến thiên tốc độ ngẫu nhiên của tài xế người trong log, nhưng khi vào vòng kín, cơ chế phản hồi trạng thái liên tục điều chỉnh gia tốc theo khoảng cách thực tế giúp xe đạt độ an toàn vượt trội với chỉ 0.14% va chạm. Ngược lại, EgoMLP có L2 vòng hở khá tốt (0.771 m) nhờ học quán tính tốc độ của chính mình, nhưng do hoàn toàn không nhìn thấy khoảng cách gap và tốc độ xe dẫn, nó không thể phản xạ khi xe trước phanh gấp, dẫn đến tỷ lệ va chạm lên tới 16.2% (trên lát cắt phanh gấp lên tới 64.3%).

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Dữ liệu vòng kín từ evidence/closedloop.json chứng minh rõ sự nguy hiểm của việc đánh giá mô hình học máy trên một lần chạy đơn lẻ. Qua 5 hạt giống huấn luyện, tỷ lệ va chạm của EgoMLP dao động dữ dội từ 2.64% (seed 9005) vọt lên tới 31.46% (seed 9007), chênh lệch tới 28.82 điểm phần trăm. Khoảng tin cậy Wilson 95% của hạt giống tốt nhất là [2.11%, 3.29%], hoàn toàn tách rời và không hề giao thoa với khoảng tin cậy của hạt giống xấu nhất là [29.79%, 33.18%]. Thậm chí PercMLP cũng dao động từ 0.49% đến 6.04% giữa các seed. Nếu một kỹ sư chỉ huấn luyện đúng 1 seed may mắn (ví dụ seed 9005 của EgoMLP với 2.64%), họ sẽ đưa ra báo cáo sai lầm rằng mô hình hoạt động tương đối an toàn, che giấu hoàn toàn tính bất ổn định nghiêm trọng trong quá trình học. Việc báo cáo kèm khoảng tin cậy Wilson và dải biến thiên qua tối thiểu 3 seed là bắt buộc để phát hiện rủi ro phân hóa hạt giống trước khi đưa hệ thống ra thử nghiệm thực tế.

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Từ evidence/agents.json, tham số khoảng cách thời gian tối ưu hiệu chỉnh được là T = 1.2 s, đạt khoảng cách Wasserstein-1 W1 = 0.357 so với phân bố thời khoảng cách của người lái thực (trung vị 1.85 s), chân thực hơn đáng kể so với tác nhân không bao giờ phản ứng có sai số W1 = 0.886.
Bảng so sánh trên lưới kịch bản cắt đầu (độ trễ 0.6 s):
- const-vel: Phát lại 77.6% | Phản ứng 77.6% (Va chạm đuôi: 0.0% / 0.0%)
- IDM: Phát lại 77.4% | Phản ứng 7.4% (Va chạm đuôi: 70.0% / 0.0%)
- EgoMLP: Phát lại 83.4% | Phản ứng 83.4% (Va chạm đuôi: 0.0% / 0.0%)
- PercMLP: Phát lại 77.7% | Phản ứng 28.4% (Va chạm đuôi: 53.4% / 0.0%)
Khi xe sau chỉ phát lại log mù quáng mà không biết phản ứng, IDM bị quy trách nhiệm cho 70.0% va chạm từ phía sau, đẩy tổng va chạm lên 77.4%. Đây hoàn toàn là va chạm giả (simulation artifact) do xe sau không phanh khi xe thử nghiệm phanh giảm tốc. Khi xe sau biết phản ứng (T = 1.2 s), va chạm đuôi của IDM giảm về đúng 0.0%, đưa tỷ lệ va chạm về mức sàn vật lý 7.4%. Nếu không có tác nhân phản ứng, môi trường mô phỏng sẽ trừng phạt chính bộ điều khiển an toàn nhất và dẫn đến kết luận thẩm định sai lệch hoàn toàn.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị cổng phát hành chặn bởi 5 quy tắc: R2 (va chạm log hạt giống xấu 6.04%, cận trên CI95 6.96% vượt ngưỡng baseline 0.14% + 1.0%), R3 (lát cắt phanh gấp 25.1%, cận trên 31.6% vượt 1.5% + 5.0%), R4 (lưới kịch bản cắt đầu 20.93% vượt 0.00% + 2.0%), R5 (độ phân tán seed chênh lệch 5.55% vượt 1.0%), và R6 (thế giới phản ứng 28.4% vượt 7.4% + 2.0%). Hai nguyên nhân gốc là: thiếu kịch bản cắt đầu ở cự ly gần trong tập dữ liệu tự nhiên (gây trượt R4, R6) và hiện tượng trôi dạt phân phối/sai số tích lũy (covariate shift) trong vòng kín (gây trượt R2, R3, R5). Mô hình được khắc phục bằng 197010 cửa sổ cắt đầu mô phỏng và 2 vòng thuật toán DAgger (dagger_windows với 895000 cửa sổ) để học cách tự sửa sai từ chính các trạng thái chệch hướng dưới nhãn chuẩn của IDM. PercMLP+ giảm va chạm log xuống 0.31% (upper 0.59%), va chạm lưới cắt đầu giảm xuống 0.44% và được cấp quyền RELEASE. Tuy nhiên, PercMLP+ KHÔNG tốt hơn IDM (va chạm log 0.31% vẫn cao hơn IDM 0.14%, lưới cắt đầu 0.44% vẫn cao hơn 0.00%) vì nó học trực tiếp từ IDM. Chưa nên phát hành ngay mà cần kiểm thử thêm tính kháng nhiễu cảm biến và thử nghiệm thực địa trên đường thử khép kín (proving ground).

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

Bài lab tồn tại ba giới hạn cốt lõi ảnh hưởng trực tiếp đến quyết định phát hành. Thứ nhất, mô phỏng chỉ xét chuyển động 1 chiều dọc (1-D longitudinal) trên đường thẳng tuyệt đối ở tần số 10 Hz với bước thời gian 0.1 s, bỏ qua hoàn toàn động học xoay thân xe, trượt ngang và gia tốc góc khi chuyển làn thực tế. Thứ hai, xe dẫn trong log không phản ứng ngược lại với hành vi của xe thử nghiệm; trong thực tế, nếu xe sau tiếp cận quá gần hoặc phát tín hiệu, tài xế xe dẫn có thể nhả ga hoặc chuyển làn tránh va chạm. Thứ ba, mô hình giả định hệ thống cảm biến hoàn hảo và cơ cấu phanh chấp hành tức thì đến 9.0 m/s^2, bỏ qua giới hạn giật (jerk limit), độ trễ đường ống thủy lực phanh và nhiễu nhận thức thực tế (như mất dấu radar hoặc nhận diện nhầm). Để thu hẹp khoảng cách tới thế giới thực, bước tiếp theo cần triển khai mô phỏng 3D đa hướng độ trung thực cao (CARLA/CarMaker), đưa vào mô hình nhiễu cảm biến và kiểm thử phần cứng trong vòng lặp (Hardware-in-the-Loop - HIL) trước khi đưa lên xe thử nghiệm sa hình.

---
*Báo cáo hoàn thành bởi học viên: Trịnh Xuân Huy (MSSV: 2A202602995)*
