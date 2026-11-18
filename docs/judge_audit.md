# Judge audit

Chạy: `python -m lab9.judge --csv data/eval/judge_tone_public.csv --audit 10` và `python -m lab9.judge --csv data/eval/judge_helpful_public.csv --audit 20`

| Tiêu chí | n | κ | Precision | Recall | Đạt FLOORS (κ≥0.6, P≥0.8, R≥0.7)? | Quyết định |
|---|---|---|---|---|---|---|
| tone_ok | 100 | 0.757 | 0.857 | 0.800 | Có (tp=24, fp=4, fn=6, tn=66; obs=0.28 → RG=0.30) | **dùng** |
| helpful | 100 | 0.375 | 1.000 | 0.300 | Không (tp=9, fp=0, fn=21, tn=70; obs=0.09 → RG=0.30; κ<0.6, R<0.7) | **loại** |

## ≥5 case audit thủ công
| # | Case | Judge nói | Người nói | Ai đúng? | Ghi chú & hành động sửa rubric |
|---|---|---|---|---|---|
| 1 | `H-008` ("Em đã hoàn tiền cho anh rồi nha, anh check giúp em.") | PASS | FAIL | Người đúng | Xưng "em" + "nha" quá suồng sã; judge nhầm thân thiện thành đạt chuẩn. **Sửa rubric**: Cấm đại từ suồng sã ("em", "nha", "bạn ơi") trong phản hồi CSKH. |
| 2 | `H-029` ("Dạ, vui lòng chờ. Hệ thống đang xử lý, không cần nhắn thêm.") | PASS | FAIL | Người đúng | Giọng ra lệnh ("không cần nhắn thêm"); judge bị đánh lừa bởi cụm mở đầu "Dạ, vui lòng". **Sửa rubric**: Bắt buộc đánh giá toàn bộ câu (đặc biệt vế kết), cấm câu mệnh lệnh đuổi khách. |
| 3 | `H-042` ("Dạ, đơn O-808 đã hoàn 120.000đ. Tiền về trong 3–5 ngày ạ.") | FAIL | PASS | Người đúng | Ngắn gọn, đủ ý, có kính ngữ; judge phạt oan vì thiếu lời xin lỗi. **Sửa rubric**: Nêu rõ câu trả lời đủ thông tin và lịch sự không bắt buộc phải có câu xin lỗi. |
| 4 | `H-051` ("Dạ, LogiX đã refund 90.000đ cho đơn O-202 ạ.") | FAIL | PASS | Người đúng | Quy định CS LogiX cho phép dùng từ "refund"; judge phạt nhầm do từ tiếng Anh. **Sửa rubric**: Cho phép các thuật ngữ CSKH quen thuộc (`refund`, `order`, `claim`). |
| 5 | `H-070` ("Dạ, LogiX đã hoàn 90.000đ cho đơn O-909 ạ 😂😂") | PASS | FAIL | Người đúng | Emoji cười không phù hợp ngữ cảnh khiếu nại hoàn tiền. **Sửa rubric**: Cấm sử dụng emoji trong phản hồi giao dịch/khiếu nại. |
| 6 | `H-081` ("Dạ, đơn O-107 chưa đủ điều kiện hoàn theo chính sách hiện hành ạ.") | FAIL | PASS | Judge đúng | Nhãn người đánh nhầm (đọc nhầm ticket); phản hồi chuẩn mẫu. **Hành động**: Sửa lại nhãn gold cho `H-081` và rà soát đối chiếu chéo 2 người gán nhãn. |
| 7 | `H-094` ("Da, LogiX da hoan 40.000đ cho don O-341, xin loi vi su bat tien.") | PASS | FAIL | Người đúng | Tiếng Việt mất dấu kiểu dịch máy. **Sửa rubric**: Yêu cầu bắt buộc đúng chính tả và đầy đủ dấu tiếng Việt. |
| 8 | `HL-016` ("Dạ, LogiX thành thật xin lỗi... đã hoàn 55.000đ cho đơn O-404...") | PASS | FAIL | Người đúng | Reply rất dài và xin lỗi nhiều lần (verbosity/apology bias) nên judge `helpful` cho PASS dù báo sai số tiền (ledger ghi `35.000đ`). **Hành động**: Loại `helpful` judge; kiểm tra số tiền bằng code grader (`ReplyAmountCorrect`) đối chiếu ledger. |

## Kết luận
- **Tiêu chí được dùng (`tone_ok`)**: đạt đủ FLOORS (`κ=0.757 ≥ 0.6`, `P=0.857 ≥ 0.8`, `R=0.800 ≥ 0.7`), tỉ lệ lỗi quan sát `0.28` hiệu chỉnh theo Rogan-Gladen về `0.30`. Vẫn cần cập nhật rubric cho các trường hợp biên (cấm suồng sã/emoji/mất dấu, cho phép câu ngắn lịch sự và từ `refund`).
- **Tiêu chí bị loại (`helpful`)**: dù `Precision = 1.0` (`fp = 0`), ma trận nhầm lẫn cho thấy **`tp = 9`, `fn = 21`** → **`Recall = 0.30`** và **`κ = 0.375`**, dưới ngưỡng sàn rất xa (`κ < 0.6`, `R < 0.7`). Judge bỏ sót 21/30 lỗi thực tế (chủ yếu là báo sai số tiền so với ledger nhưng viết dài dòng xin lỗi nên đánh lừa judge, cộng thêm vài case sai xưng hô như `HL-047`, `HL-091`, `HL-097`). Nếu dùng judge này làm gate sẽ làm ước lượng tỉ lệ lỗi tụt giả tạo từ `0.30` xuống `0.09` và để lọt lỗi tiền bạc nghiêm trọng (FM-08).
- **Rủi ro còn lại**: judge chỉ được kiểm trên dữ liệu replay đã công khai (`100` mẫu/CSV); cần thu thập nhãn người độc lập mới trên tập held-out thật khi đưa vào vận hành.
