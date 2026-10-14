# Phase 1 — Baseline và tiêu chí đo

Ngày thực hiện: 2026-10-14
Repo: `K4-DAY26-VuMinhHoang-2A202602371-Colosseum-Agent-Arena-Kit`

## Kết luận checkpoint

**Checkpoint 1 chưa đạt đầy đủ; baseline kit/fixture có thể tái lập đã được lập.** `spar.py` yêu cầu world export thật và dừng trước khi bắt đầu trận vì `kit/world/<world_id>/manifest.json` chưa tồn tại. Do đó không có HP, trace chiến đấu, credits từng round hoặc kết quả chiến thuật thật nào được ghi nhận cho Rookie/Operator/Adversary trong lượt này. Không nội suy những số đó từ mô tả trong README.

## Môi trường và kiểm tra đầu vào

- Python 3.12.10, pytest 8.3.4.
- World export: không có; `kit/world/` chỉ chứa mã nguồn.
- `py -3.12 spar.py --bot rookie --as all --rounds 2 --seed 1` dừng với thông báo cần world export. Hai bot còn lại dùng cùng cơ chế spar nên cũng đang bị chặn bởi điều kiện này.
- `py -3.12 -m kit.gate_no_key`: **PASS**, 96 file quét, 0 vi phạm.
- Đã chạy nhóm kiểm tra không cần thao tác filesystem tạm: `py -3.12 -m pytest tests/test_prosecute.py tests/test_loop.py tests/test_cost.py tests/test_spar.py tests/test_anchor.py -k "not find_real_corpus_root_is_independent_of_clone_depth" --tb=short -q -p no:cacheprovider --basetemp .pytest-tmp2`: **4065 passed, 2 skipped, 1 deselected**.
- Full suite không thể đánh giá trong sandbox hiện tại: nhiều test cần tạo/duyệt/xóa thư mục tạm và gặp `PermissionError` khi truy cập temp path. Các lỗi đó không được quy kết là lỗi logic của sản phẩm.

## Baseline theo nhiệm vụ

### Defend — `agent/`

Demo `py -3.12 -m agent.gateway` chạy thành công. Bốn lệnh mẫu (provenance, query, A2A delegation, catalog list) đều trả `forward`; deny path tồn tại nhưng demo chỉ gọi thủ công. Đây là baseline **forward gần như mọi thứ**, đúng mô tả starter, chưa phải defense an toàn.

Các giới hạn đã thấy trong mã và demo:

- Gateway chưa thực hiện đầy đủ ROUTE / ADMIT / AUTHORIZE / BUDGET.
- Strategy helper đã có mask/cost, catalog trap, tool successor, cache, pacing, replica/delegation policy, nhưng chưa được nối thành policy gateway hoàn chỉnh.
- Grounding là guardrail thực; injection scanner, redaction và arithmetic verifier là stub.
- Prompt là một phần policy sống; telemetry chỉ hỗ trợ debug của phía mình, không phải bằng chứng đối thủ/referee.

### Attack — `deck/` và bot đối thủ

Không thể chạy mutation spar nếu thiếu world. Baseline kiến trúc theo README của kit:

| Bot | Baseline được mô tả trong repo | Số đo thực ở checkpoint này |
|---|---|---|
| Rookie | Forward mọi lệnh, mask rộng, hay gọi catalog, bỏ lease; README mô tả thường cạn 100 credits khoảng round 3 | Chưa đo trong trận |
| Operator | Có pin/diff và traceparent nhưng thiếu verification; nhầm identity với authority; README mô tả false positive trên blank khoảng 25% | Chưa đo trong trận |
| Adversary | Kiểm tra card/skill/audience/act, pin/re-verify, exactly-once và budget thích ứng; README mô tả false positive blank dưới 5% | Chưa đo trong trận |

Các con số trong cột mô tả là tuyên bố/tóm tắt của README, không phải kết quả spar được tái lập ở máy này.

### Prosecute — `eval/`

`py -3.12 eval/prosecute.py` chạy trên 40 fixture prosecution:

- Lỗi: **0**; timeout trên 5 giây: **0**.
- Filed 2 claims; adjudicated 2; verified 2; unproven 0; false 0; rejected 0.
- Precision **1.000**, recall **0.059**, F1 **0.111**, false-claim rate **0.000**.
- `enforcement_failure`: bắt được **2/2** fixture (recall 1.00). 16 hook lớp lỗi khác vẫn chưa triển khai.

Đây là số liệu baseline chính xác, có thể tái lập độc lập với world thật.

## Phát hiện từ các demo starter

- `py -3.12 -m agent.gateway`: PASS; gateway khởi tạo command/decision được và forward toàn bộ bốn lệnh demo.
- `py -3.12 -m eval.prosecute`: PASS như số liệu trên.
- `py -3.12 -m agent.strategy`: demo hiện có mâu thuẫn assertion. Với TOOL_SPECS thật, chi phí tính ra là 9 credits cho round disciplined và 49 cho careless; sau 10 round disciplined, còn 10 credits, không phá sản, nhưng demo lại assert rằng phải phá sản ở round 10 và thất bại (`bankrupt_by() is None`). Khi chạy file trực tiếp `py -3.12 agent/strategy.py`, import fallback cho ra careless=34 rồi fail assertion `careless >= 45`; cách chạy đúng như module sử dụng spec thật và làm lộ assertion stale.
- `py -3.12 -m agent.guardrails`: phần kiểm tra grounding đầu chạy được, nhưng demo dừng tại assert phân loại anchor sai định dạng. `Anchor.parse` trả chuỗi đó vào bucket khác với `malformed` mà demo đang kỳ vọng. Đây là lỗi/inconsistent assumption trong demo starter cần xem lại ở Phase 2.
- Chạy file gateway trực tiếp (`py -3.12 agent/gateway.py`) không phù hợp do import package; dùng module `py -3.12 -m agent.gateway`.

## Tiêu chí đo được chốt để dùng ở các phase sau

1. **Gateway:** số command forward/deny/rewrite; lệnh trái quyền bị chặn; lỗi Decision/exception; số lần dùng tool deprecated; protocol violations.
2. **Economy:** tổng và credits còn lại qua 10 rounds; credits/call so với tool cost; tránh mask `*`/catalog trap; giữ budget cho late rounds.
3. **Answer:** đúng nội dung theo truth khi fixture cho phép; số citation/anchor có thật và đã retrieve trong exchange; injection/privacy/unsupported precision được xử lý; tỷ lệ abstain hợp lý.
4. **Prosecutor:** precision/recall/F1/false rate, claim rejected, lỗi/timeout, và recall từng lớp trên `score_prosecutor`.
5. **Deck:** validation thật trên world, attack có trigger và mutation tác động đúng ASK, kết quả spar và phản ứng blank.

## Lệnh tái lập khi có world export

```powershell
# Windows PowerShell
py -3.12 spar.py --bot rookie --as all --seed 1
py -3.12 spar.py --bot operator --as all --seed 1
py -3.12 spar.py --bot adversary --as all --seed 1
```

```bash
# macOS / Linux / Ubuntu
python3.12 spar.py --bot rookie --as all --seed 1
python3.12 spar.py --bot operator --as all --seed 1
python3.12 spar.py --bot adversary --as all --seed 1
```

Trước khi chạy, xác nhận world có `kit/world/<world_id>/manifest.json` và không có `truth.json`. Với tái lập có kiểm soát, giữ nguyên seed, rounds và role, rồi lưu trọn stdout/run log để so sánh.

## Việc cần gỡ để đóng checkpoint hoàn toàn

1. Lấy world export thật từ giảng viên/Release và đặt đúng cấp thư mục.
2. Chạy lại spar với ba bot; ghi HP, credits, missed claims và trace của mỗi run.
3. Dùng đường dẫn temp khả dụng để chạy các test phụ thuộc filesystem; sandbox hiện chặn một số temp directory nên không coi full-suite run lỗi quyền là kết luận về code.
4. Sau khi baseline được lưu, sửa hai demo assertion stale trong Phase 2 theo hành vi/spec hiện hành, không sửa `kit/`.
