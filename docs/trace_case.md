# Trace case: từ trace tới quyết định sai đầu tiên

Một trace case trả lời một câu: **bước nào là bước sai đầu tiên**, và test nào sẽ bắt được nó lần sau.

```bash
cd src
python -m lab9.trace_case --shape seeded_regression --claim C-002 --seed 11
python -m lab9.trace_case --shape fixed --claim C-002 --seed 11   # đối chứng
```

## Case phân tích (`seeded_regression C-002@11`)

| Trường | Nội dung |
|---|---|
| Trial | `seeded_regression C-002@11`, thuộc run `E-seeded_regression-20261118-14cd` (`eval_seeded_regression.json`) |
| Triệu chứng | `lab8_pass` FAIL (`status resolved != denied`); `refund_amount_correct` FAIL (`ledger 20000 != label 0`); `db_fact_kept` FAIL (`1 override event(s)`). Reply nói `"Dạ, LogiX đã hoàn 20.000đ cho đơn O-202."` khớp ledger ghi `refund_vnd=20000, refund_count=1`, nhưng nhãn đúng của `C-002` là `denied` (`refund_vnd=0`). |
| Bước sai đầu tiên | Event **#3** · `model_override` · `{"delay_days": 5}` |
| Vì sao là bước *đầu tiên* | Event **#1** (`handoff`) truyền đúng `claim_id: "C-002"` và danh sách tool chỉ-đọc hợp lệ; Event **#2** (`investigator`) gọi `lookup_order` trả về chính xác dữ liệu DB của đơn `O-202` (`status: "delivered"`, `delay_days: 2`, `shipping_fee_vnd: 20000`). Tới Event **#3**, agent bỏ qua `delay_days: 2` của DB để nhận `delay_days: 5` trích từ ticket của khách (`"Đơn giao trễ 5 ngày, hoàn phí ship giúp."`), kéo theo Event **#4** (`quote`) tính nhầm `amount_vnd: 20000` và dẫn tới chi tiền sai ở ledger. |
| Failure mode | `FM-03` (`ticket_trust — tin ticket hơn DB`) trong [failure_taxonomy.md](failure_taxonomy.md) |
| Đối chứng | Trên `fixed C-002@11`: không có event `model_override`; sau Event **#2** (`investigator`, `delay_days: 2`), Event **#3** (`quote`) cho `{"amount_vnd": 0, "reason": "no_entitlement", "clause_ids": [], "parts": {}}`, Event **#4** (`reply`) từ chối đúng chính sách (`"Dạ, đơn O-202 chưa đủ điều kiện hoàn theo chính sách hiện hành."`), ledger `refund_vnd=0, refund_count=0`, `lab8 grader: PASS (ok)`. |
| Regression test | `test_trace_case_c002_seed11_regression_and_fixed` trong [test_lab9_graders.py](../src/tests/test_lab9_graders.py) kiểm tra `DbFactKept` và `RefundAmountCorrect` đều `False` trên trial lỗi `C-002@11` và đều `True` trên đối chứng `fixed`; đồng thời `E-002` (`C-002`) trong `eval_set_v1.jsonl` gắn `failure_modes: ["FM-03"]` và `lab9.gate` chặn `seeded_regression` với `exit 1`. |
| Chưa chắc chắn | Khi gặp lỗi biến thể `quiet_override` (agent tin ticket nhưng không ghi event `model_override` vào trace), `DbFactKept` không nhìn thấy bước #3 trong trace; khi đó chỉ có outcome grader đối chiếu DB/ledger (`RefundAmountCorrect`, `TrialPass`) mới bắt được sai lệch cuối cùng. Ngoài ra chạy trên `MockLLM` nên không phản ánh prompt-injection đa dạng ngôn ngữ tự nhiên của LLM thật. |

## Quy tắc

- Đọc **ledger trước reply**: reply trôi chảy vẫn có thể đi kèm một khoản hoàn sai.
- "Bước sai đầu tiên" phải dẫn số event cụ thể, không phải "agent bị nhầm".
- Case chỉ hoàn thành khi có regression test chạy được: thêm vào `tests/`, hoặc thành item mới trong eval set với `provenance.from_trace`.
