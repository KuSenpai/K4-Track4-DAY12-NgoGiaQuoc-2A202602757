# REPORT — Lab Day 12

> Điền **bằng lời và số liệu của chính bạn** (lấy từ màn hình và từ `evidence/*.json`). Mỗi câu 4–8 dòng là đủ. Số không có trong bằng chứng của bạn sẽ bị trừ điểm. Báo cáo công khai trên GitHub; giảng viên đối chiếu với `evidence/` và với bài của các bạn khác.

| | |
|---|---|
| Họ và tên | `<điền>` |
| MSSV (đúng bằng `--student-id` bạn đã chạy) | `2A202602757` |
| GitHub | `<điền: tên tài khoản>` |
| Seed chia dữ liệu / seed mô hình (xem `evidence/closedloop.json`, mục `student`) | split seed 4419 / model seeds 9702–9706 |
| Số seed huấn luyện bạn dùng (`--seeds`) | 5 |

## Kết quả cuối

Điền từ `evidence/closedloop.json`, `openloop.json`, `gate.json` (cho mỗi mô hình học, lấy **dải qua các seed**).

| Bộ điều khiển | L2 vòng hở (m) | Va chạm vòng kín trên log (%) | Quyết định cổng |
|---|---|---|---|
| constant-velocity | 1.258 | 20.2 | BLOCK (R2, R3, R4, R6) |
| IDM | 1.550 | 0.1 | (baseline) |
| EgoMLP | 0.752–0.760 (TB 0.756) | 10.6–28.7 | BLOCK (R2, R3, R4, R5, R6) |
| PercMLP | 0.610–0.619 (TB 0.615) | 0.2–0.8 | BLOCK (R2, R3, R4, R6) |
| PercMLP+ (sau khi sửa) | 0.808 (±0.016) | 0.1–0.2 | RELEASE (qua 7/7) |

## Câu 1 · Dữ liệu
Bước 0 bạn phát hiện những vấn đề gì ở dữ liệu? Vì sao không dùng cột `Acc_LV`, và bạn thay bằng gì? Bao nhiêu quỹ đạo bị loại vì trùng, và vì sao chia theo quỹ đạo?

Có ba vấn đề. (1) Cột `Acc_LV` gần như không liên quan đến đạo hàm tốc độ xe dẫn: tương quan chỉ 0.045, trong khi cột tương ứng của xe theo sau là 0.929. Vì vậy em không dùng `Acc_LV` mà dùng gia tốc tự tính từ `Speed_LV` đã làm mượt (`a_lead`); nếu dùng cột thô thì việc khai thác ca phanh sẽ bắt nhầm nhiễu. (2) Có 383 quỹ đạo trùng hệt nhau (9993 → 9610 quỹ đạo sạch), và chúng nằm ở các id cách xa nhau, nên chia ngẫu nhiên dễ để một bản ở train, một bản ở test → rò rỉ, điểm test đẹp giả. (3) Khung hình trong cùng một quỹ đạo tương quan rất mạnh (10 Hz), nên nếu chia theo khung thì mô hình đã "thấy" gần như cùng tình huống lúc huấn luyện. Do đó phải chia theo **quỹ đạo**: không quỹ đạo nào có mặt ở cả train lẫn test.

## Câu 2 · Phát lại
Bản refactor float32 khác bit ở bao nhiêu đoạn, vượt dung sai ở bao nhiêu? Thay đổi AEB làm lệch bao nhiêu đoạn, và vì sao lệch ở đúng những khung đó (mở một đoạn ra xem)? Phát theo thời gian thật làm lộ ra điều gì mà phát nhanh nhất che đi?

Chạy lại cùng mã: 0/600 đoạn khác. Bản refactor float32 khác bit ở **600/600** đoạn nhưng **0** đoạn vượt dung sai (atol = rtol = 1e-3, sai khác lớn nhất 4.0e-4 m/s²), vì phép cộng số thực không kết hợp: (0.1+0.2)+0.3 = 0.6000000000000001 ≠ 0.6. So `==` sẽ báo lỗi giả, nên cần dung sai cỡ 1e-3 m/s², nhỏ hơn nhiều so với mức lệnh có ý nghĩa vật lý. AEB làm lệch **7/600** đoạn (179, 193, 205, 307, 313, 336, 566). Mở đoạn 179: chỉ các khung 65–67 lệch, gap 3.41 → 3.02 m, closing ≈ 2.2 m/s, **TTC 1.47 → 1.41 s < 1.5 s**, lệnh đổi từ −5.71 thành −6.0 m/s²; đoạn 193 cũng vậy (khung 87–88, TTC 1.49 và 1.46 s). AEB chỉ kích hoạt khi TTC < 1.5 s nên chỉ những khung đó đổi. Phát nhanh nhất: 0 khung trễ, 0 khung rớt vì không có khái niệm hạn chót. Phát theo thời gian thật: 582 khung trễ và 577 khung rớt trên 59616 khung, làm 370/600 đoạn đổi lệnh. Tức là phát nhanh nhất che mất lỗi trễ hạn mà xe thật sẽ gặp.

## Câu 3 · Kịch bản
Lưới của bạn (tốc độ cruise, giới hạn khoảng cách, số ô): bao nhiêu % ô **không ai tránh được**? Độ trễ phản ứng 0 → 0,6 s làm vùng đâm của IDM tăng bao nhiêu? Log có phủ vùng khó không? (dẫn giới hạn phanh mạnh nhất của người lái trong log, và phần lưới đòi hỏi hơn thế)

Lưới ở tốc độ cruise 10 m/s, khoảng cách 2.76–59.3 m (p2–p98 của log), tốc độ tiến gần 0–10 m/s, 60 × 45 = 2700 ô. Biên bất khả thi gap < dv·t_react + dv²/(2·9) chiếm **1.0 %** ô khi t_react = 0 s và **5.1 %** khi t_react = 0.6 s. Bộ mô phỏng khớp hoàn toàn với biên này: 0 ô lệch ở cả hai độ trễ. Vùng đâm của IDM tăng từ 1.0 % lên 5.1 %, nhưng trùng **đúng** vùng bất khả thi, nên phần "IDM tự gây ra" (attributable) là 0.0 % ở cả hai mức trễ: IDM chỉ đâm khi không ai tránh được. Ngược lại, const-vel đâm 73.4 % lưới, tức 68.2 % là lỗi của chính nó. Log **không** phủ vùng khó: người lái mạnh nhất chỉ phanh 3.6 m/s² (p99.9), trong khi 5.2 % lưới đòi phanh mạnh hơn mức đó. 614 ca xe dẫn phanh khai thác được (6.4 % quỹ đạo) chỉ có peak decel p50 −2.5 và p99 −4.2 m/s², nên các chấm trắng tập trung ở vùng dễ.

## Câu 4 · Vòng hở và vòng kín
Dán bảng L2 và tỷ lệ va chạm của bốn bộ điều khiển (kèm hạng ở mỗi cách chấm). Cặp nào đổi thứ tự? Vì sao IDM có L2 như vậy mà vẫn lái tốt? Vì sao EgoMLP có L2 tốt hơn constant-velocity mà vẫn đâm?

| Bộ điều khiển | L2 vòng hở | Hạng | Va chạm vòng kín (TB seed) | Hạng |
|---|---|---|---|---|
| const-vel | 1.258 m | 3 | 20.2 % | 4 |
| IDM | 1.550 m | 4 | 0.1 % | 1 |
| EgoMLP | 0.756 m | 2 | 19.5 % | 3 |
| PercMLP | 0.615 m | 1 | 0.5 % | 2 |

Ba cặp đổi thứ tự: (const-vel, IDM), (IDM, EgoMLP), (IDM, PercMLP). IDM có L2 tệ nhất (1.550 m) vì nó không bắt chước phong cách của từng người lái mà giữ khoảng cách an toàn theo luật riêng; L2 phạt mọi khác biệt so với người, kể cả khác biệt an toàn hơn. Nhưng IDM luôn phản ứng với gap và tốc độ tiến gần, nên trong vòng kín chỉ đâm 0.1 % (4/2883). EgoMLP chỉ nhìn lịch sử tốc độ của chính nó, **không thấy khoảng cách hay tốc độ xe dẫn**. Trên log, người lái thường đi đều nên đoán "tiếp tục như trước" cho L2 tốt (0.756 m), nhưng khi xe dẫn phanh thì nó không có tín hiệu nào để phanh theo: 58.0 % va chạm trên các quỹ đạo xe dẫn phanh. Lát cắt phanh chỉ là 4.2 % số cửa sổ, nên L2 trung bình che mất lỗi đó.

## Câu 5 · Một lần chạy chưa đủ
EgoMLP dao động bao nhiêu giữa các seed? Khoảng tin cậy Wilson nói gì? Nếu bạn chỉ chạy đúng một seed thì có thể báo kết quả nào, và vì sao điều đó nguy hiểm?

Qua 5 seed, tỷ lệ va chạm của EgoMLP dao động từ **10.6 %** (seed 9702, 306/2883) đến **28.7 %** (seed 9705, 826/2883), gấp gần 3 lần, trong khi L2 vòng hở gần như không đổi (0.752–0.760 m). Khoảng Wilson 95 % của seed tốt nhất là [9.5, 11.8] %, của seed xấu nhất là [27.0, 30.3] %. Hai khoảng **không chồng nhau**, nên sự khác biệt đến từ seed huấn luyện chứ không phải nhiễu lấy mẫu trên 2883 quỹ đạo. Nếu chỉ chạy một seed, ta có thể báo "EgoMLP đâm 10.6 %" hoặc "28.7 %" tuỳ may rủi. Cả hai đều có khoảng tin cậy hẹp nên trông rất chắc chắn, nhưng thực ra không đại diện cho mô hình. Vì vậy cổng R2 dùng seed **xấu nhất**, và R5 yêu cầu dải giữa các seed ≤ 1 điểm phần trăm; EgoMLP rớt R5 (10.6..28.7 %).

## Câu 6 · Tác nhân phản ứng
Dán bảng "xe sau phát lại" và "xe sau phản ứng". `T` bạn hiệu chỉnh là bao nhiêu, xe sau chân thực đến mức nào so với xe không phản ứng? Vì sao IDM bị tính "va chạm giả" khi xe sau chỉ phát lại, và điều này làm đổi kết luận về bộ điều khiển ra sao?

Tỷ lệ ô lưới cắt vào có va chạm bất kỳ (trước hoặc sau), t_react = 0.6 s:

| Bộ điều khiển | Xe sau phát lại | Xe sau phản ứng | Va chạm sau: phát lại / phản ứng |
|---|---|---|---|
| const-vel | 73.4 % | 73.4 % | 0.0 % / 0.0 % |
| IDM | 70.1 % | 5.1 % | 64.9 % / 0.0 % |
| EgoMLP | 71.7 % | 71.7 % | 0.0 % / 0.0 % |
| PercMLP | 87.3 % | 19.1 % | 69.3 % / 0.0 % |

T được hiệu chỉnh là **1.4 s** (khoảng cách Wasserstein-1 tới phân bố headway của người là 0.35, người có headway trung vị 1.86 s). Xe không bao giờ phản ứng có W1 = 0.88, tức kém chân thực hơn rõ rệt. Khi xe sau chỉ phát lại log, nó không biết xe thử vừa phanh, nên IDM, vốn phanh đúng lúc khi bị cắt vào, bị xe sau đâm vào đuôi ở 64.9 % lưới. Đó là **va chạm giả** do thế giới không phản ứng, không phải lỗi của IDM. Trong thế giới phát lại, IDM (70.1 %) trông tệ ngang const-vel (73.4 %), vì const-vel không phanh nên không bị đâm sau. Với xe sau phản ứng, va chạm sau về 0 % và IDM còn 5.1 %, đúng bằng vùng bất khả thi. Kết luận đảo ngược hẳn: bộ phanh đúng không còn bị phạt.

## Câu 7 · Cổng phát hành và sửa
Quy tắc nào chặn PercMLP và nguyên nhân gốc là gì (dẫn số)? Bạn sửa bằng gì và vì sao nó chữa đúng nguyên nhân? Sau khi qua cổng, PercMLP+ có **tốt hơn** IDM không? Nó có nên được phát hành không, và bạn đề xuất làm gì trước khi quyết định?

PercMLP bị chặn bởi R2, R3, R4, R6 (qua R5 vì dải seed chỉ 0.2..0.8 %). Nguyên nhân thứ nhất là **kịch bản log không chứa**: trên lưới cắt vào, PercMLP tự gây va chạm ở 14.0 % lưới (R4, baseline 0.0 % + 2), và 19.1 % với xe sau phản ứng (R6, baseline 5.1 %). Log không có ca bị cắt vào ở cự ly hẹp hay phanh mạnh hơn 3.6 m/s². Nguyên nhân thứ hai là **sai số tích luỹ** khi tự lái: trên log, seed xấu nhất đâm 0.8 % với cận trên 1.2 % > 0.1 % + 1 (R2), và 2.9 % trên lát phanh với cận trên 6.5 % (R3). Mô hình trôi vào những trạng thái người lái chưa từng đi qua. Cách sửa: thêm 197090 cửa sổ từ 20000 ca cắt vào mô phỏng (chữa nguyên nhân 1) và 2 vòng DAgger, gắn nhãn IDM tại chính các trạng thái mô hình tự đi vào (chữa nguyên nhân 2). PercMLP+ qua 7/7 quy tắc: 0.1..0.2 % va chạm trên log (cận trên 0.5 %), lưới 0.1 %, thế giới phản ứng 5.3 % so với 5.1 %. Tuy vậy, nó chỉ **không kém hơn** IDM, không tốt hơn: va chạm 0.2 % so với 0.1 % của IDM, và nó học nhãn từ chính IDM nên khó vượt thầy. L2 vòng hở còn tệ đi (0.615 → 0.808 m), cho thấy điểm vòng hở không đo được điều quan trọng. Ở cruise 20 m/s chưa thấy khi huấn luyện, PercMLP+ vẫn tự gây đâm 0.6..4.1 % lưới, trong khi IDM là 0.0 %. Em chưa đề xuất thay IDM, mà nên chạy shadow mode hoặc thêm kịch bản ngoài phân bố (tốc độ cao, nhiễu cảm biến, độ trễ) trước khi quyết định.

## Câu 8 · Bài lab này không chứng minh được gì
Chọn **ít nhất ba** giới hạn quan trọng nhất đối với việc quyết định phát hành và giải thích vì sao mỗi cái làm kết luận của bạn kém chắc chắn. Điều gì cần làm tiếp để thu hẹp khoảng cách đến thế giới thật?

(1) **Thế giới mô phỏng là giả định của chúng ta**: chỉ một chiều dọc, động lực học lý tưởng, phanh tối đa 9 m/s², không có trượt hay mặt đường ướt. Các con số "0 ô lệch" hay "PercMLP+ 0.1 % lưới" chỉ đúng trong mô hình vật lý đó. (2) **Không có cảm biến thật**: gap và tốc độ xe dẫn được cho gần như hoàn hảo. Thử nhiễu gap ±1 m, tốc độ ±0.5 m/s đã đẩy PercMLP lên 0.2..0.9 %, và thực tế còn có mất dấu, trễ, nhận diện sai. (3) **Xe dẫn trong log không phản ứng với xe thử**, và kịch bản giới hạn trong phân bố log (cruise 10 m/s, phanh người ≤ 3.6 m/s²). Ở 20 m/s chưa thấy khi huấn luyện, PercMLP+ đã tự đâm tới 4.1 % lưới, nên vùng ngoài phân bố vẫn chưa được kiểm. (4) **Mẫu thống kê hữu hạn**: chỉ 174 quỹ đạo xe dẫn phanh và 5 seed. Khoảng tin cậy ở mức 0.1 % vẫn rộng so với yêu cầu an toàn (cỡ lỗi trên triệu km). Cần làm tiếp: kiểm thử với mô hình cảm biến và độ trễ thật, thêm kịch bản ngang và nhiều làn, các tác nhân phản ứng đa dạng, nhiều seed và nhiều dữ liệu hơn, rồi chạy shadow mode trên xe thật trước khi phát hành.
