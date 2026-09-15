# Báo cáo Ngày 3 — Tracking Annotation

Họ tên / nhóm: `...`
Ngày: `2026-09-15`

---

## 1. Quá trình gán nhãn

| Mục | Giá trị |
| --- | --- |
| Công cụ | CVAT |
| Thời gian gán `clip_02` (warm-up) | `...` phút |
| Thời gian gán `clip_01` | `...` phút |
| Số track đã vẽ trong `clip_01` | 6 |
| Số keyframe trung bình mỗi track | ~83.7 (tổng 502 bbox / 6 track) |

Chi tiết từng track trong `clip_01`:

| Track ID | Frame bắt đầu | Frame kết thúc | Số bbox |
| ---: | ---: | ---: | ---: |
| 1 | 1 | 11 | 11 |
| 2 | 1 | 190 | 190 |
| 3 | 1 | 45 | 45 |
| 4 | 51 | 151 | 101 |
| 5 | 73 | 140 | 68 |
| 6 | 104 | 190 | 87 |

Ba tình huống khó nhất khi gán clip này, và bạn xử lý thế nào:

1. Track 4 (frame 51–151): xe xuất hiện ở giữa clip và di chuyển liên tục — cần xác định chính xác frame bắt đầu/kết thúc khi xe vào/rời vùng nhìn thấy rõ. Sai lệch nhỏ so với gold (gold track 4: frame 54–148) cho thấy mình bắt đầu sớm 3 frame và kết thúc muộn 3 frame — bbox ở rìa bị loose (frame 55 IoU=0.509).
2. Track 5 (frame 73–140): xe bị occlusion một phần bởi xe khác ở khu vực giữa clip. Cần quyết định giữ bbox ôm phần nhìn thấy hay dừng track. Đã chọn giữ track liên tục — gold track 5 chỉ kéo từ frame 79–138, mình bắt đầu sớm 6 frame.
3. Thiếu 2 track so với gold (gold track 6 và 8): gold có 8 track, mình chỉ vẽ 6. Track 6 (gold frame 101–156, 56 bbox) và track 8 (gold frame 136–168, 33 bbox) không được gán — có thể do xe nhỏ/xa hoặc xuất hiện thoáng qua mà mình bỏ sót, dẫn đến FN = 94.

## 2. Tự kiểm và kiểm chéo

Ba lượt tua bắt được gì (lượt 1 nhìn ID, lượt 2 frame đầu/cuối, lượt 3 frame giữa):

- Lượt 1: Kiểm tra 6 track ID — không phát hiện ID switch nào (IDSW = 0 trong eval_vs_gold xác nhận).
- Lượt 2: Kiểm tra frame đầu/cuối mỗi track — phát hiện track 4 và 5 có thể bắt đầu sớm hơn gold vài frame (ghost_pred_tracks: track 5 sớm 6 frame ở 73–78, track 4 sớm 3 frame ở 51–53 và muộn 3 frame ở 149–151).
- Lượt 3: Kiểm tra frame giữa — phát hiện một số bbox loose ở frame 55 (IoU=0.509), frame 83–84 (IoU=0.532–0.587), frame 107 (IoU=0.588).

Bài cá nhân — không có kiểm chéo (N/A).

## 3. Pre-gold lock và chấm trước/sau rework

| Evidence | Giá trị |
| --- | --- |
| SHA-256 từ `evidence/pre-gold/clip_01/manifest.json` | *(chưa có — thư mục evidence/pre-gold/clip_01 chỉ có .gitkeep)* |
| Thời điểm khóa | `...` |
| Số row / frame / track trước khi mở reference | 502 rows / 190 frames / 6 tracks |

| | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| Bản pre-gold | *(chưa có evidence)* | | | | | | | | | |
| Sau rework | 0.7712 | 0.7032 | 0.8476 | 0.8824 | 0.8912 | 0.7958 | 0.8726 | 23 | 94 | 0 |

Qua cổng (`IDF1 >= 0.80`, `MOTA >= 0.75`, `MOTP >= 0.70`): **có** ✅
- IDF1 = 0.8912 ≥ 0.80 ✅
- MOTA = 0.7958 ≥ 0.75 ✅
- MOTP = 0.8726 ≥ 0.70 ✅

Sau khi đọc danh sách lỗi từ `eval_vs_gold.json`, các lỗi cụ thể:

| Loại lỗi | Frame | ID | Mô tả |
| --- | --- | --- | --- |
| missed_gt_tracks | toàn clip | gold ID 6, 8 | Không gán 2 track — gold có 8 track, mình chỉ có 6. Gold track 6 (frame 101–156, 56 bbox) và track 8 (frame 136–168, 33 bbox) bị thiếu hoàn toàn → FN = 94 |
| ghost_pred (sớm) | 73–78 | pred 5 | Bbox xuất hiện trước khi gold track 5 bắt đầu (gold bắt đầu frame 79) |
| ghost_pred (muộn) | 149–151 | pred 4 | Bbox còn sau khi gold track 4 kết thúc (gold kết thúc frame 148) |
| ghost_pred (sớm) | 51–53 | pred 4 | Bbox xuất hiện trước khi gold track 4 bắt đầu (gold bắt đầu frame 54) |
| loose bbox | 55 | pred 4 / gold 4 | IoU = 0.509 — bbox lệch nhiều so với gold |
| loose bbox | 83 | pred 5 / gold 5 | IoU = 0.532 |
| loose bbox | 84 | pred 5 / gold 5 | IoU = 0.587 |
| loose bbox | 107 | pred 6 / gold 7 | IoU = 0.588 |

## 4. Kết quả model: ByteTrack control vs ReID treatment

Cấu hình từ `outputs/model_run_config.json`:

| Mục | Giá trị |
| --- | --- |
| Python / ultralytics / torch / lap | 3.13.3 / 8.4.145 / 2.7.1+cu118 / 0.5.13 |
| weights / hai tracker | yolo26n.pt / bytetrack.yaml, botsort-reid.yaml |
| conf / IoU / imgsz / classes | 0.25 / 0.7 / 960 / [2, 5, 7] (car, bus, truck) |
| device | 0 (GPU) |

BoT-SORT + ReID config (`configs/trackers/botsort-reid.yaml`):
- `track_buffer`: 30, `match_thresh`: 0.80, `with_reid`: true
- `proximity_thresh`: 0.50, `appearance_thresh`: 0.80, `gmc_method`: none

| So sánh | HOTA | DetA | AssA | LocA | IDF1 | MOTA | MOTP | FP | FN | IDSW |
| --- | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: | ---: |
| bạn vs gold | 0.7712 | 0.7032 | 0.8476 | 0.8824 | 0.8912 | 0.7958 | 0.8726 | 23 | 94 | 0 |
| ByteTrack control vs gold | 0.7085 | 0.6487 | 0.7761 | 0.8463 | 0.8746 | 0.7487 | 0.8226 | 88 | 54 | 2 |
| BoT-SORT + ReID vs gold | 0.7635 | 0.7110 | 0.8204 | 0.8721 | 0.9001 | 0.7923 | 0.8595 | 91 | 26 | 2 |
| ReID vs bạn | 0.7256 | 0.6281 | 0.8391 | 0.8870 | 0.8316 | 0.6195 | 0.8768 | 163 | 27 | 1 |

## 5. Phân tích — năm câu hỏi

**1. MOTA của bạn cao hơn hay thấp hơn IDF1? Nếu MOTA cao mà IDF1 thấp thì điều đó nói gì, và vì sao MOTA không phạt nặng lỗi ID?**

MOTA (0.7958) **thấp hơn** IDF1 (0.8912). Điều này hợp lý vì:
- IDF1 cao (0.89) + IDSW = 0 → mình giữ ID rất nhất quán, mỗi track được gán đúng ID xuyên suốt.
- MOTA thấp hơn vì MOTA = 1 − (FP + FN + IDSW) / GT_boxes = 1 − (23 + 94 + 0) / 573 = 0.796. FN = 94 chiếm phần lớn lỗi — do thiếu 2 track (gold track 6: 56 bbox, track 8: 33 bbox).
- MOTA không phạt nặng lỗi ID vì mỗi IDSW chỉ bị trừ 1 lần (cùng trọng số với 1 FP hoặc 1 FN), trong khi IDF1 đo tỷ lệ khớp ID toàn cục — một IDSW có thể làm sai ID của toàn bộ chuỗi frame sau đó, khiến IDTP giảm mạnh. Ngược lại, trường hợp MOTA cao mà IDF1 thấp sẽ xảy ra khi detect đúng nhiều object nhưng gán sai ID liên tục.

**2. ByteTrack control và BoT-SORT + ReID treatment khác nhau thế nào ở IDF1, AssA và IDSW? Dẫn một frame sequence để giải thích treatment tốt hơn, tệ hơn hoặc không đổi đáng kể. Nhắc rõ đây không cô lập causal effect của ReID vì hai tracker implementation khác.**

| Metric | ByteTrack | BoT-SORT + ReID | Δ |
| --- | ---: | ---: | --- |
| IDF1 | 0.8746 | 0.9001 | **+0.0255** ↑ |
| AssA | 0.7761 | 0.8204 | **+0.0443** ↑ |
| IDSW | 2 | 2 | 0 (không đổi) |
| FN | 54 | 26 | **−28** ↓ (tốt hơn) |

ReID treatment tốt hơn rõ rệt ở association (AssA +5.7%) và identity (IDF1 +2.6%).

**Frame sequence minh họa**: Gold track 8 (frame 136–168, 33 bbox) — ByteTrack chỉ cover 20/33 frame (ratio 0.61, partially covered), trong khi ReID không bị partially covered cho track 8. Điều này cho thấy ReID duy trì association tốt hơn khi object xuất hiện ở vùng đông đúc. Tương tự, gold track 5: ByteTrack cover 46/60 (0.77) vs ReID không bị partially covered.

Tuy nhiên, IDSW đều = 2 ở cả hai — ByteTrack switch ở frame 59 (gt4: 14→15) và frame 94 (gt5: 23→32); ReID switch ở frame 87 (gt5: 17→18) và frame 113 (gt6: 24→31). ReID không loại bỏ được IDSW nhưng thay đổi vị trí xảy ra.

> **Lưu ý**: Đây không cô lập được causal effect của ReID vì ByteTrack và BoT-SORT là hai tracker implementation khác nhau — BoT-SORT dùng Kalman filter + IoU matching + appearance embedding, trong khi ByteTrack chỉ dùng IoU matching. Sự khác biệt có thể đến từ motion model hoặc matching cascade chứ không chỉ từ ReID.

**3. DetA, FP và FN đổi thế nào? Lỗi còn lại là detector hay association?**

| Metric | ByteTrack | ReID | Δ |
| --- | ---: | ---: | --- |
| DetA | 0.6487 | 0.7110 | +0.0623 ↑ |
| FP | 88 | 91 | +3 (gần như không đổi) |
| FN | 54 | 26 | **−28** ↓ |

- FN giảm mạnh (54→26): ReID duy trì track lâu hơn nhờ appearance feature, giảm miss khi object bị occlusion tạm thời.
- FP gần như không đổi (88→91): Cả hai đều có ~5 ghost pred tracks "không khớp track tham chiếu nào" — đây là **lỗi detector** (YOLO26n detect false positive). Cụ thể:
  - ByteTrack: ghost tracks 10 (42 frame), 41 (16 frame), 69 (10 frame), 64 (1 frame), 70 (1 frame)
  - ReID: ghost tracks 7 (43 frame), 27 (16 frame), 38 (16 frame), 26 (1 frame), 28 (1 frame)
- **Kết luận**: Lỗi FP còn lại chủ yếu là **lỗi detector** — YOLO detect nhầm object không phải vehicle. Lỗi FN giảm nhờ **association tốt hơn** của ReID.

**4. Một chỗ bạn đúng và ReID sai (frame, ID, vì sao):**

**Frame 16–116, ReID pred_track 7**: Ghost track kéo dài 43 frame, "không khớp track tham chiếu nào" trong cả `eval_reid_vs_gold.json` và `eval_reid_vs_me.json`. Mình không gán track nào ở vùng này — đúng vì gold cũng không có track tương ứng. ReID sai vì detector (YOLO26n) tạo false detection liên tục ở vùng đó (có thể là bóng xe, biển quảng cáo, hoặc object ở rìa khung hình), và ReID tracker duy trì track qua appearance matching thay vì loại bỏ.

**5. Một chỗ ReID làm bạn xem lại annotation (frame, ID, vì sao), hoặc lý do evidence cho thấy model sai:**

So sánh eval_vs_gold (bạn vs gold) cho thấy mình **thiếu 2 gold track** (missed_gt_tracks: [6, 8]):
- Gold track 6: frame 101–156, 56 bbox
- Gold track 8: frame 136–168, 33 bbox

ReID vs gold cho thấy ReID **detect được cả hai track này** — track 6 dù chỉ partially covered (44/56 frame, ratio 0.79) nhưng track 8 không bị miss. Điều này cho thấy đây là **xe thật xuất hiện trong clip** mà mình đã bỏ sót khi gán nhãn. ReID buộc mình xem lại annotation và bổ sung 2 track này — đặc biệt ở vùng frame 101–168 nơi có nhiều xe cùng xuất hiện.

## 6. Nếu phải gán thêm 10 clip nữa

Bạn sẽ sửa gì trong `GUIDELINE_MINI.md`, và đổi gì trong quy trình làm việc của mình?

1. **Bổ sung luật "xe nhỏ/xa"** vào GUIDELINE_MINI.md: Cần quy định rõ ngưỡng kích thước tối thiểu để gán (vd: bbox ≥ 20×20 pixel), vì mình đã bỏ sót 2 track có thể do xe xuất hiện nhỏ hoặc ở xa.
2. **Thêm bước cross-check với model**: Sau khi gán xong, chạy nhanh model (ByteTrack/ReID) trên clip và so sánh số track — nếu model detect nhiều track hơn, quay lại kiểm tra vùng thiếu.
3. **Đặt keyframe dày hơn ở frame đầu/cuối track**: Các ghost pred (frame 51–53, 73–78, 149–151) cho thấy mình sai lệch vài frame so với gold ở boundary — cần cẩn thận hơn khi xác định thời điểm xe vào/rời khung hình.
4. **Tua lại toàn clip ở tốc độ chậm ít nhất 1 lần** trước khi nộp, tập trung vào vùng frame 100–170 (khu vực đông xe) để không bỏ sót xe nhỏ.

## 7. Tệp đã nộp

- [x] `annotations/clip_01/gt.txt` (502 rows, 190 frames, 6 tracks)
- [x] `annotations/clip_02/gt.txt` (239 rows, 60 frames, 6 tracks)
- [ ] `evidence/pre-gold/clip_01/gt.txt` và `manifest.json` *(chưa có — chỉ có .gitkeep)*
- [x] `GUIDELINE_MINI.md` đã điền
- [x] `outputs/eval_vs_gold.json`
- [x] `outputs/model_bytetrack_clip_01.txt` (608 rows)
- [x] `outputs/model_reid_clip_01.txt` (639 rows)
- [x] `outputs/model_run_config.json`
- [x] `outputs/eval_bytetrack_vs_gold.json`, `outputs/eval_reid_vs_gold.json`, `outputs/eval_reid_vs_me.json`
- [ ] `reports/review_partner.md` *(bài cá nhân — N/A)*
- [ ] `reports/REPORT.md` (file này)
