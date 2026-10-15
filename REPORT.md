# REPORT — Lab Day 12

|                                                                                           |                    |
| ----------------------------------------------------------------------------------------- | ------------------ |
| Họ và tên                                                                              | `Ngô Gia Quốc` |
| MSSV (đúng bằng `--student-id` bạn đã chạy)                                      | `2A202602757`    |
| GitHub                                                                                    | `KuSenpai`       |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | 4419 / 9702–9706  |
| Số seed huấn luyện bạn dùng (`--seeds`)                                            | 5                  |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển       | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
| ----------------------- | ---------------- | --------------------------------- | ------------------- |
| constant-velocity       | 1.258            | 20.2                              | BLOCK               |
| IDM                     | 1.550            | 0.1                               | (baseline)          |
| EgoMLP                  | 0.752–0.760     | 10.6–28.7                        | BLOCK               |
| PercMLP                 | 0.610–0.619     | 0.2–0.8                          | BLOCK               |
| PercMLP+ (sau khi sửa) | 0.808            | 0.1–0.2                          | RELEASE             |

## Câu 1 · Dữ liệu

Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

- `Acc_LV` gần như là nhiễu: tương quan với đạo hàm tốc độ chỉ 0.045 (cột của xe sau là 0.929). Em dùng gia tốc tính từ `Speed_LV` đã làm mượt.
- Có 383 quỹ đạo trùng hệt nhau (9993 → 9610) nằm ở các id xa nhau. Chia ngẫu nhiên sẽ lọt bản sao sang tập test và làm điểm đẹp giả.
- Chia theo quỹ đạo vì các khung trong cùng một quỹ đạo gần như giống nhau. Chia theo khung thì mô hình đã "thấy" tình huống test lúc huấn luyện.

## Câu 2 · Phát lại

Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

- Refactor float32 khác bit ở 600/600 đoạn nhưng 0 đoạn vượt dung sai (lệch tối đa 4.0e-4 m/s²). Phép cộng số thực không kết hợp, nên phải so có dung sai chứ không dùng `==`.
- AEB làm lệch 7/600 đoạn. Ở đoạn 179 chỉ khung 65–67 lệch, đúng lúc TTC giảm 1.47 → 1.41 s, tức dưới ngưỡng 1.5 s; lệnh đổi từ −5.71 thành −6.0 m/s².
- Phát nhanh nhất cho 0 khung trễ và 0 khung rớt. Phát thời gian thật cho 582 khung trễ và 577 khung rớt, làm 370/600 đoạn đổi lệnh. Phát nhanh nhất che mất lỗi trễ hạn.

## Câu 3 · Kịch bản

Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

- Lưới: cruise 10 m/s, gap 2.76–59.3 m, 2700 ô. Vùng bất khả thi là 1.0 % ô (trễ 0 s) và 5.1 % ô (trễ 0.6 s); bộ mô phỏng khớp 0 ô lệch.
- Vùng đâm của IDM tăng 1.0 % → 5.1 %, trùng đúng vùng bất khả thi. Phần lỗi do chính IDM gây ra là 0.0 %, còn const-vel là 68.2 %.
- Log không phủ vùng khó: người lái phanh mạnh nhất 3.6 m/s², trong khi 5.2 % lưới đòi phanh mạnh hơn. 614 ca phanh khai thác được có p99 chỉ −4.2 m/s².

## Câu 4 · Vòng hở và vòng kín

Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 (m) | Hạng | Va chạm (%) | Hạng |
| ----------------- | ------ | ----- | ------------ | ----- |
| const-vel         | 1.258  | 3     | 20.2         | 4     |
| IDM               | 1.550  | 4     | 0.1          | 1     |
| EgoMLP            | 0.756  | 2     | 19.5         | 3     |
| PercMLP           | 0.615  | 1     | 0.5          | 2     |

- Ba cặp đổi thứ tự: (const-vel, IDM), (IDM, EgoMLP), (IDM, PercMLP).
- IDM có L2 tệ nhất vì không bắt chước người lái, nhưng luôn phản ứng theo gap nên chỉ đâm 0.1 %.
- EgoMLP không nhìn thấy xe dẫn. Trên log, người lái thường đi đều nên L2 tốt, nhưng khi xe dẫn phanh thì nó đâm ở 58.0 % số ca. Lát cắt phanh chỉ chiếm 4.2 % cửa sổ nên L2 trung bình che mất lỗi này.

## Câu 5 · Một lần chạy chưa đủ

EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

- Va chạm của EgoMLP dao động 10.6 % → 28.7 % qua 5 seed, trong khi L2 gần như không đổi.
- Khoảng Wilson 95 % của seed tốt nhất [9.5, 11.8] % và của seed xấu nhất [27.0, 30.3] % không chồng nhau, nên khác biệt này do seed chứ không phải nhiễu.
- Nếu chỉ chạy một seed, ta có thể báo 10.6 % với khoảng tin cậy hẹp, trông rất chắc chắn nhưng sai. Vì vậy cổng dùng seed xấu nhất và R5 yêu cầu dải ≤ 1 điểm phần trăm.

## Câu 6 · Tác nhân phản ứng

Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

| Bộ điều khiển (trễ 0.6 s) | Xe sau phát lại | Xe sau phản ứng | Va chạm sau: phát lại / phản ứng |
| ------------------------------ | ----------------- | ----------------- | ------------------------------------- |
| const-vel                      | 73.4 %            | 73.4 %            | 0.0 / 0.0 %                           |
| IDM                            | 70.1 %            | 5.1 %             | 64.9 / 0.0 %                          |
| EgoMLP                         | 71.7 %            | 71.7 %            | 0.0 / 0.0 %                           |
| PercMLP                        | 87.3 %            | 19.1 %            | 69.3 / 0.0 %                          |

- T hiệu chỉnh là 1.4 s (W1 = 0.35), sát với người lái hơn nhiều so với xe không phản ứng (W1 = 0.88).
- Xe sau phát lại không biết IDM vừa phanh nên đâm vào đuôi ở 64.9 % lưới. Đây là va chạm giả, không phải lỗi của IDM.
- Trong thế giới phát lại, IDM (70.1 %) trông tệ ngang const-vel (73.4 %). Khi xe sau phản ứng, IDM chỉ còn 5.1 %, đúng bằng vùng bất khả thi.

## Câu 7 · Cổng phát hành và sửa

Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

- PercMLP rớt R2, R3, R4, R6. Có hai nguyên nhân gốc:
  - Log thiếu kịch bản cắt vào: tự đâm 14.0 % lưới (R4) và 19.1 % lưới khi xe sau phản ứng (R6).
  - Sai số tích luỹ khi tự lái: cận trên 1.2 % > 0.1 % + 1 điểm (R2).
- Cách sửa: thêm 197090 cửa sổ cắt vào mô phỏng (chữa nguyên nhân 1) và 2 vòng DAgger gắn nhãn IDM tại chính các trạng thái mô hình tự đi vào (chữa nguyên nhân 2). PercMLP+ qua 7/7 quy tắc với va chạm 0.1–0.2 % và lưới 0.1 %.
- PercMLP+ chỉ **không kém hơn** IDM chứ không tốt hơn: 0.2 % so với 0.1 %, và nó học từ chính IDM. L2 còn tệ đi (0.615 → 0.808 m), và ở 20 m/s chưa thấy khi huấn luyện nó vẫn tự đâm tới 4.1 % lưới.
- Chưa nên thay IDM. Nên chạy shadow mode và thử thêm kịch bản ngoài phân bố trước khi quyết định.

## Câu 8 · Bài lab này không chứng minh được gì

Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

1. **Mô phỏng 1 chiều, vật lý lý tưởng** (phanh tối đa 9 m/s², không trượt): kết quả như 0 ô lệch hay 0.1 % lưới chỉ đúng trong mô hình này.
2. **Không có cảm biến thật:** chỉ cần nhiễu gap ±1 m đã đẩy PercMLP lên 0.9 %. Ngoài đời còn có mất dấu và độ trễ.
3. **Kịch bản giới hạn trong phân bố log** (cruise 10 m/s, người lái phanh ≤ 3.6 m/s²). Ở 20 m/s, PercMLP+ đã đâm tới 4.1 % lưới.
4. **Mẫu nhỏ:** chỉ 174 ca xe dẫn phanh và 5 seed, quá ít so với yêu cầu an toàn thật.

Việc cần làm tiếp: dùng mô hình cảm biến và độ trễ thật, thêm kịch bản nhiều làn, tăng dữ liệu và số seed, rồi chạy shadow mode trên xe.
