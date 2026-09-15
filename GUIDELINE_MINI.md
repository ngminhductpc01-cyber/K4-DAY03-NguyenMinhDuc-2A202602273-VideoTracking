# Mini annotation guideline — Ngày 3 (tracking)

> Điền file này **trong lúc gán nhãn**, không phải sau khi xong. Mỗi lần bạn dừng
> lại nghĩ "cái này tính sao nhỉ?" thì đó là một dòng phải ghi vào đây.
>
> Đây là tài liệu mà người gán nhãn tiếp theo sẽ đọc để làm giống bạn. Nếu hai
> người trong nhóm gán khác nhau, gần như luôn là vì file này chưa nói rõ — chứ
> không phải vì ai kém.

Nhóm / tên: *(bài cá nhân)*
Clip: `clip_01`, `clip_02`

---

## 1. Phạm vi: gán cái gì, không gán cái gì

Một lớp duy nhất: **`vehicle`** — xe bốn bánh (xe con, van, xe buýt, xe tải).

| Gán | Không gán |
| --- | --- |
| xe con, SUV, taxi, xe bán tải | người đi bộ |
| van, minivan | xe đạp |
| xe buýt, minibus | **xe máy / mô tô** |
| xe tải, xe đầu kéo | xe trong ảnh quảng cáo, trong gương, dưới bóng nước |

Bổ sung của nhóm (nếu có): Không gán xe đang đỗ ngoài rìa ảnh nếu không di chuyển và diện tích nhìn thấy < 20×20 pixel. Gán tất cả xe bốn bánh đang di chuyển hoặc đang dừng tạm (đèn đỏ) dù xa hay gần — bài học từ việc bỏ sót gold track 6 và 8.

## 2. Luật ID — phần quan trọng nhất

| Tình huống | Luật của nhóm | Vì sao |
| --- | --- | --- |
| Xe bị che một phần rồi hiện lại | giữ nguyên ID nếu bị che **dưới 25 frame** (mặc định của lab: 25 frame = 2 giây @ 12.5 fps) | Ngưỡng 25 frame đủ để cover occlusion tạm thời bởi xe khác đi ngang qua; eval cho thấy IDSW = 0 khi giữ luật này |
| Xe bị che lâu hơn ngưỡng trên | Tạo **track mới** với ID mới, không cố nối lại | Không có cách xác nhận chắc chắn đó là cùng một xe sau > 2 giây; tránh gán sai ID |
| Xe rời khung hình rồi quay lại | mặc định: **track mới** | Xe rời khung hình = kết thúc track. Nếu quay lại, không thể chắc chắn là cùng xe → tạo ID mới để tránh IDSW |
| Hai xe cắt nhau / chồng lên nhau | Giữ nguyên ID cho cả hai; bbox ôm phần nhìn thấy được của từng xe; nếu xe bị che hoàn toàn thì dừng track tạm, chờ hiện lại trong ≤ 25 frame | Eval cho thấy loose bbox ở frame 83–84 (IoU 0.53–0.59) khi hai xe gần nhau — cần cẩn thận vẽ bbox ôm đúng phần thấy được |

## 3. Luật bbox

| Tình huống | Luật của nhóm |
| --- | --- |
| Xe bị cắt bởi rìa ảnh | bbox chạm đúng rìa, không đoán phần ngoài ảnh |
| Xe bị xe khác che một phần | bbox ôm phần **nhìn thấy được** — không ôm cả phần bị che |
| Xe vừa xuất hiện, còn rất nhỏ / rất mờ | Bắt đầu track từ frame đầu tiên xác định được là xe bốn bánh; ngưỡng: bbox ≥ ~20×20 pixel. Lưu ý: gold bắt đầu track 4 ở frame 54, track 5 ở frame 79 — không nên bắt đầu sớm hơn khi xe còn mờ (bài học từ ghost pred ở frame 51–53 và 73–78) |
| Xe đang đỗ, không di chuyển | Gán track bình thường nếu xe bốn bánh rõ ràng, kể cả khi đứng yên (vd: chờ đèn đỏ). Xe đỗ ngoài rìa ảnh nhỏ hơn 20×20 pixel thì bỏ qua |
| Keyframe đặt dày ở đâu | Đặt dày (mỗi 1–2 frame) ở: (1) frame đầu/cuối track khi xe vào/rời khung hình, (2) khi xe bị occlusion hoặc đi gần xe khác, (3) khi xe đổi hướng. Đặt thưa (mỗi 5–10 frame) khi xe di chuyển thẳng đều. Bài học: sai lệch 3–6 frame ở boundary so với gold gây ghost pred |

## 4. Ít nhất ba ca mơ hồ đã gặp thật

Ghi **frame cụ thể** và **ID cụ thể**, không ghi chung chung.

### Ca 1
- Clip / frame / ID: clip_01 / frame 51–53 / Track 4
- Tình huống: Xe bắt đầu xuất hiện ở rìa khung hình, còn rất nhỏ và một phần bị cắt. Khó xác định frame chính xác nên bắt đầu gán track.
- Quyết định: Bắt đầu track từ frame 51 — sớm hơn gold 3 frame (gold bắt đầu frame 54).
- Lý do: Đã nhìn thấy xe nhưng gold đợi đến khi xe rõ hơn. Eval cho thấy 3 frame đầu là ghost pred, và bbox ở frame 55 có IoU chỉ 0.509 (loose). **Bài học**: nên đợi xe rõ ràng hơn trước khi bắt đầu track — khoảng frame 54 trở đi.

### Ca 2
- Clip / frame / ID: clip_01 / frame 73–78 / Track 5
- Tình huống: Xe xuất hiện ở xa, đang tiến lại gần. Khó xác định đây là xe bốn bánh hay chỉ là shape mờ. Bắt đầu track sớm 6 frame so với gold (gold bắt đầu frame 79).
- Quyết định: Gán từ frame 73. Eval cho thấy 6 frame đầu (73–78) là ghost pred "đã có bbox trước khi track tham chiếu xuất hiện".
- Lý do: Xe tuy nhìn thấy nhưng gold yêu cầu chắc chắn hơn. **Bài học**: với xe ở xa, đợi thêm vài frame cho đến khi xe đủ lớn (~30×30 pixel) và có hình dáng xe rõ ràng.

### Ca 3
- Clip / frame / ID: clip_01 / frame 101–168 / Gold track 6 và 8 (bị bỏ sót)
- Tình huống: Vùng frame 101–168 có nhiều xe cùng xuất hiện. Hai xe (gold track 6: frame 101–156, 56 bbox; gold track 8: frame 136–168, 33 bbox) đã bị bỏ sót hoàn toàn — missed_gt_tracks trong eval_vs_gold.
- Quyết định: Không gán → thiếu 2 track, FN = 94.
- Lý do: Có thể do xe nhỏ, ở xa, hoặc bị che khuất bởi xe khác nên không nhận ra. ReID model detect được cả hai track này (eval_reid_vs_gold). **Bài học**: tua lại vùng đông xe ở tốc độ chậm, zoom vào kiểm tra kỹ từng xe — đặc biệt frame 100–170.

## 5. Sửa gì sau khi chấm với gold và sau khi kiểm chéo

Luật nào trong file này hoá ra còn thiếu hoặc còn mơ hồ? Viết lại cho rõ:

- **Thiếu luật về xe nhỏ/xa**: Trước đó không có ngưỡng rõ ràng → bỏ sót 2 track. Đã bổ sung: gán tất cả xe bốn bánh nhìn thấy được dù xa hay gần, chỉ bỏ qua nếu diện tích < 20×20 pixel và xe không di chuyển. Khi nghi ngờ, nên gán — false positive ít hại hơn false negative (FP chỉ thêm 23 bbox, nhưng miss track gây FN = 94).
- **Thiếu luật về frame boundary**: Không rõ "khi nào bắt đầu track" → bắt đầu sớm 3–6 frame, tạo ghost pred. Đã bổ sung: bắt đầu track khi xe có hình dáng rõ ràng và bbox ≥ 20×20 pixel; kết thúc khi xe không còn nhận diện được. Tham khảo gold: track 4 bắt đầu frame 54 (không phải 51), track 5 bắt đầu frame 79 (không phải 73).
- **Thiếu quy trình kiểm tra cuối**: Cần thêm bước tua toàn clip ở tốc độ chậm (0.5x) tập trung vào vùng đông xe trước khi nộp, để tránh bỏ sót xe.
