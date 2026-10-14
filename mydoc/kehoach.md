# Kế hoạch hoàn thành Lab Colosseum Agent Arena Kit

## Mục tiêu và phạm vi

Hoàn thành đủ ba nhiệm vụ trong mỗi vòng đấu của lab Day 26 (MCP/A2A Infrastructure & Agentic Routing):

1. **Attack (`deck/`)**: xây ASK và mutation deck có tác động thật, hợp lệ và khó phòng thủ.
2. **Defend (`agent/`)**: định tuyến/quyết định MCP-A2A an toàn; tạo câu trả lời grounded, trích dẫn đúng và tuân thủ guardrail.
3. **Prosecute (`eval/`)**: phát hiện lỗi của đối thủ, nộp claim đúng lớp với bằng chứng cụ thể, trong quota.

Đầu ra nộp là bundle được seal gồm `agent/`, `deck/`, `eval/`. Không sửa `kit/`, `bots/`, `fixtures/`; chỉ dùng standard library trong ba thư mục được sở hữu. Deadline là lúc bắt đầu phiên đấu: `agent/` và `deck/` bị khóa khi phiên bắt đầu, còn `eval/` tiếp tục được hoàn thiện trong phiên.

## Trạng thái repo lúc lập kế hoạch

- Repo có README, RULES, starter agent/deck/prosecutor và bộ test công khai.
- Chưa có `mydoc/`; thư mục này sẽ được tạo để lưu kế hoạch.
- Chưa có dữ liệu world thật dưới `kit/world/<world_id>/`; chỉ có mã loader/fixture. World thật là phụ thuộc cần lấy từ Release/giảng viên và kiểm tra không có `truth.json`.
- Deck starter được mô tả là deck đã xác minh trên corpus thật; cần hiểu, mô phỏng và điều chỉnh theo chiến lược đội thay vì mặc định giữ nguyên.
- `agent/gateway.py` có bốn điểm chính cần triển khai/wire: ROUTE, ADMIT, AUTHORIZE, BUDGET. `agent/strategy.py` có các helper policy; `agent/guardrails.py` có kiểm tra grounding hoạt động nhưng scanner injection, redaction và kiểm chứng số học còn stub.
- `eval/prosecute.py` đã có detector `enforcement_failure`; các hook cho 16 lớp còn lại chưa hoạt động. Bộ fixture prosecution đã được cung cấp.
- Không tìm thấy hướng dẫn `AGENTS.md` trong danh sách file repo.

## Nguyên tắc triển khai

- Đọc `RULES.md` và các README trong `agent/`, `deck/`, `eval/`, `kit/world/` trước khi sửa.
- `Gateway.decide` phải đồng bộ, thuần, không I/O, không network/thread/sleep, trả `Decision` hợp lệ trong 250 ms. Không có `execute`; gateway chỉ quyết định, arena thực thi.
- Không đoán khi thiếu căn cứ: deny/abstain là chiến lược hợp lệ; deny không tốn credit. Không deny vô điều kiện vì blank card phạt false positive.
- MCP: gọi tool kế nhiệm (`slides.query` thay `slides.search`), dùng field mask hẹp, tuân lease và partial/continuation, quản lý If-Match/idempotency với write, kiểm tra lỗi/rate limit và kiểm tra `lang`/replica.
- Mọi citation phải thuộc anchors thực sự nhận được trong exchange hiện tại. Chỉ dùng thông tin cần trả lời; không để nội dung Note/peer biến thành chỉ thị; không lộ dữ liệu riêng.
- Prosecutor chỉ nộp claim khi có thể chỉ ra event/span/anchor chứng minh. Tối đa 4 claim, tối đa 1 claim mỗi family; claim sai mất `0.8 × weight`.
- Chạy lệnh từ Makefile đúng như repo: `make test`, `make validate`, `make spar ...`, `make submit TEAM=<tên>`. `make qualify` đã retired; không dùng làm gate.

## Phase 0 — Chốt hợp đồng và chuẩn bị môi trường

### Công việc

1. Đọc trọn `README.md`, `RULES.md`, `agent/README.md`, `deck/README.md`, `eval/README.md`, `kit/world/README.md`.
2. Lập bản đồ interface và luồng trace: `kit/loop/agent.py`, `agent/gateway.py`, `kit/mcp/types.py`, `kit/mcp/specs.py`, `kit/mcp/servers.py`, referee/rubric, `validate_deck.py`.
3. Tạo môi trường Python 3.12 theo hướng dẫn (`make install`) và chạy readiness/kiểm tra ban đầu.
4. Lấy world từ Releases hoặc giảng viên theo README; giải nén đúng cấp `kit/world/<world_id>/manifest.json`, kiểm tra hash nếu có và xác nhận không có `truth.json`. Sau đó chạy `make doctor`.
5. Nếu world chưa đến, tiến hành các phần độc lập bằng fixture; ghi rõ kiểm tra corpus/deck thực chưa thể hoàn tất.

### Checkpoint 0 — Sẵn sàng

- Hiểu ownership và các lệnh thực tế; không có thay đổi dưới thư mục cấm.
- `make test` chạy được trên fixture.
- Có world thật hợp lệ, `make doctor` xanh; hoặc đã ghi nhận rõ phụ thuộc đang chờ và không tuyên bố deck đã được xác thực trên corpus.

## Phase 1 — Baseline và tiêu chí đo

### Công việc

1. Chạy `make spar BOT=rookie`, `make spar BOT=operator AS=all` và một lượt với `adversary` để đọc trace/UI và hiểu baseline.
2. Chạy các demo/module baseline cho gateway, strategy, guardrails, prosecutor; lưu số credits, quyết định, lỗi, citation và kết quả fixture làm mốc so sánh.
3. Đọc các bài test liên quan (`tests/test_*`) để hiểu input/output được pin và các tình huống edge.
4. Chia các invariant cần đạt: gateway không lỗi/không vượt quyền; câu trả lời đúng và có căn cứ; citation traceable; prosecutor precision cao, claim hợp quota; deck hợp lệ và attack kích hoạt.

### Checkpoint 1 — Baseline có thể tái lập

- Có ghi chú baseline cho 3 vai trò, chi phí/call và các stub còn thiếu.
- Biết lệnh tái chạy từng luồng; đã phân biệt lỗi implementation với thiếu world thật.

## Phase 2 — Phòng thủ: strategy và gateway (`agent/`)

### Công việc

1. Hoàn thiện `agent/strategy.py` theo helper có sẵn: cost/mask, tránh catalog trap, deprecated successor, pacing/credit floor, cache theo anchor+mask, delegation chỉ khi tăng giá trị và rate/budget cho phép, chọn replica theo dữ liệu drift.
2. Wiring helper vào bốn job ở `Gateway.decide`:
   - **ROUTE**: server/tool/replica đúng, ưu tiên tool hiện hành; rewrite chỉ khi `ToolCall` hợp lệ và không đổi ý định ngoài policy.
   - **ADMIT**: chặn command sai schema/unknown tool, partial hoặc request không cần thiết theo policy; không blanket-deny.
   - **AUTHORIZE**: xác minh `act`, `aud`, scope và quan hệ giữa command với context; đặc biệt bảo vệ write/A2A, không coi identity hợp lệ là authority.
   - **BUDGET**: tính cost theo spec, mask và trạng thái credits; deny/rewrite khi không đáng chi.
3. Giữ mọi nhánh `decide` total và an toàn: input edge case không làm raise; mọi `Decision` đúng schema; lý do deny rõ và ngắn.
4. Hoàn thiện guardrail:
   - yêu cầu citation theo ASK và grounding trên tool result của lượt hiện tại;
   - phát hiện chỉ thị giả trong retrieved content và A2A response;
   - redaction nội dung riêng trước khi tạo answer;
   - kiểm tra số liệu có căn cứ, không tạo precision vượt nguồn;
   - xử lý xung đột/replica disagreement và abstain khi thiếu căn cứ.
5. Cập nhật `agent/prompt.md` để model tuân budget/iteration, chọn tool/mask hẹp, kiểm tra partial/lease/replica, xử lý conflict và trả lời ngắn kèm anchors; prompt bổ trợ harness, không thay thế nó.
6. Giữ telemetry chỉ phục vụ debug; không coi telemetry do agent phát là bằng chứng referee.

### Checkpoint 2 — Gateway và guardrail an toàn

- Mọi đường đi trả về Decision hợp lệ trong giới hạn đồng bộ/thuần; không có quyền vượt `ctx.act`/scope.
- Có chính sách cụ thể cho lease, partial result, provenance/If-Match, idempotency, retry, rate limit, ngôn ngữ, replica và giá.
- Citation lạ/malformed, không retrieve ở exchange hiện tại, output injection hoặc dữ liệu riêng được xử lý an toàn; trường hợp chưa đủ chứng cứ thì abstain.
- So với baseline Rookie: không còn hành vi forward mù, nhưng vẫn trả lời được khi request hợp lệ.

## Phase 3 — Deck tấn công (`deck/`)

### Công việc

1. Đọc schema/vocabulary trong `deck/README.md` và `validate_deck.py`; xác nhận 14 lá: 10 attacks + 4 blanks, lineup 10 lượt không lặp.
2. Kiểm kê lớp/layer và deck starter; giữ đủ tối thiểu 3 MCP, 3 A2A, 2 gateway, 6 lớp khác nhau. Chọn đúng 9 duel class/op/target.
3. Với từng attack, ghi giả thuyết: invariant bị phá, mutation thực sự tác động vào trường cần thiết của ASK, event nào sẽ chứng minh, cách defender có thể phát hiện. Ưu tiên các cặp mutation/ASK làm thay đổi ground truth thay vì chỉ tạo nhiễu vô hại.
4. Mọi `replica_flip`/`swap_replica` phù hợp phải dùng `path_id` thuộc drift set thật. Không bịa anchor/peer/Note.
5. Chọn trigger và thứ tự lineup để giảm lặp layer, tạo áp lực hợp lý; cân nhắc blank như cách kiểm tra false-positive thay vì chỉ chọn tấn công tối đa.
6. Kiểm tra `defense_event` đúng literal `gateway.denied` theo hướng dẫn starter deck; kiểm tra target/op/value và ask fields bắt buộc.
7. Chạy `make validate` trên world thật và sửa tất cả FAIL. Đọc WARN về lethality band; validator kit-only không thể tự chứng minh đầy đủ đấu Rookie/adversary, vì vậy cần kiểm chứng qua spar nếu engine/ladder được cung cấp.

### Checkpoint 3 — Deck khóa được

- `make validate` không FAIL trên world thật; đủ luật deck/lineup/layer/class/drift/anchor.
- Có bảng card → mutation → invariant → bằng chứng dự kiến; không có attack vô hiệu do ASK không nhạy với mutation.
- Deck có kết quả spar: Rookie bị đánh trúng hợp lý, adversary có thể phòng thủ; đã xem xét false positive trên blank.
- `deck/` sẵn sàng khóa trước giờ bắt đầu.

## Phase 4 — Công tố và đo detector (`eval/`)

### Công việc

1. Đọc schema claim, grammar evidence refs, families/weights, quota và detector `enforcement_failure` đang có.
2. Dùng fixtures được gán nhãn để cài detector theo từng lớp; mỗi detector phân biệt positive với near-miss, chỉ dùng trace/answer/card được cấp, không dựa vào source opponent.
3. Trích evidence refs chính xác (`evt`, `answer.span`, `anchor`) và ghi `expected`, `observed`, `argument` ngắn đúng invariant; bảo đảm đủ proof refs, không gom nhiều class vào một claim.
4. Thực thi `ProsecutionBudget`: tối đa 4 claim/tối đa 1 family; ưu tiên claim có xác suất chứng minh cao, không shotgunning lớp nặng.
5. Đo bằng `score_prosecutor` trên toàn bộ fixture: precision, recall, F1, false rate, per-class recall, errors/timeouts/rejected. Tối ưu precision trước để không chịu recoil; rồi mở rộng recall bằng detector có bằng chứng chắc.
6. Rà hiệu năng dưới 5 giây, xử lý event thiếu/biến dạng, trường hợp không có claim và thứ tự claim không làm vượt quota.

### Checkpoint 4 — Prosecutor có thể tin cậy

- Không exception/timeout trên fixture; không claim bị reject do schema/quota.
- Claim positive chỉ có khi đủ bằng chứng causal; near-miss không bị kết luận nhầm.
- Báo cáo score/per-class đã xem xét; không đánh đổi false claim chỉ để tăng số lượng cáo buộc.

## Phase 5 — Tích hợp và spar lặp (`agent/`, `deck/`, `eval/`)

### Công việc

1. Chạy `make test` và `make validate`; phân loại mọi lỗi theo module/invariant, không sửa kit/bots/fixtures.
2. Chạy spar theo thứ tự dễ đến khó:
   - `make spar BOT=rookie AS=defender`
   - `make spar BOT=operator AS=all`
   - `make spar BOT=adversary AS=defender`
   - `make spar BOT=adversary AS=prosecutor`
   - `make spar BOT=operator AS=all` để kiểm tra vòng tích hợp.
3. Với từng trace, xem đủ 5 giai đoạn: QUESTION, ACTION, ANSWER/citation, EVAL/proof, REFEREE/HP. Đối chiếu claim với event causal và answer với anchors thực nhận.
4. Theo dõi credits/iteration, partial/continuation, lease còn hạn, retry, rate limit, replica flips, A2A identity, write precondition/idempotency, privacy/injection.
5. Mỗi vòng chỉ sửa một nhóm nguyên nhân, chạy lại cùng tình huống rồi xác nhận không hồi quy fixture hoặc deck legality.

### Checkpoint 5 — Sẵn sàng phiên đấu

- Public tests và validate đạt; không có integrity/schema/timeout hoặc vi phạm hợp đồng.
- Qua Rookie ở defense; có kết quả có thể giải thích trước Operator/Adversary; prosecutor có claim chứng minh và không bị phạt false.
- Credits đủ cho các vòng cuối, câu trả lời grounded/guardrailed, deck order đã chốt.

## Phase 6 — Đóng băng và nộp bundle

### Trước khi phiên bắt đầu

1. Chốt `agent/` và `deck/` trước deadline khóa; lưu bản sao/commit theo quy trình đội.
2. Chạy kiểm tra cuối: `make test`, `make validate`, `make doctor` (nếu có world thật).
3. Đảm bảo chỉ sửa `agent/`, `deck/`, `eval/`; không có thư viện ngoài, key/model client, code bị cấm hoặc dữ liệu `truth.json`.
4. Nộp bằng `make submit TEAM=<tên-đội>`; xác nhận bundle tại `submissions/<tên-đội>.bundle` có đủ ba thư mục và manifest/hash.

### Trong phiên đấu

- Không chỉnh `agent/` hoặc `deck/` sau khi khóa. Tiếp tục tinh chỉnh `eval/` nếu quy chế/runner cho phép.
- Mỗi lượt ghi lại card, tool trace, answer anchors, claims, outcome/referee và credits để điều chỉnh quyết định prosecution còn được phép.

### Checkpoint 6 — Nộp hoàn tất

- Bundle được tạo và kiểm tra; biết rõ vị trí bundle.
- Có ghi chú kết quả cuối: test/validate, score prosecutor, các lượt spar tiêu biểu, rủi ro còn lại do corpus/runner hoặc WARN validator.

## Bảng checkpoint nhanh

| Mốc | Điều kiện qua |
|---|---|
| C0 — Sẵn sàng | Python 3.12, public tests; world thật được xác nhận hoặc thiếu phụ thuộc được ghi nhận |
| C1 — Baseline | Có kết quả Rookie/Operator/Adversary và danh sách vấn đề định lượng |
| C2 — Defense | Gateway total/pure/valid; authorization, routing, budget và guardrail đã wire |
| C3 — Attack | Deck/lineup hợp lệ trên world thật; từng attack có hiệu lực và bằng chứng dự kiến |
| C4 — Prosecution | Fixture scoring không lỗi/timeout/reject; claim chính xác và có proof refs |
| C5 — Tích hợp | Tests/validate/spar đạt, credits và câu trả lời còn ổn định dưới áp lực |
| C6 — Nộp | `make submit TEAM=...` sinh bundle đầy đủ trước/đúng thời hạn |

## Danh sách việc ưu tiên ngay

1. Cài môi trường, đọc nốt interface/test theo map ở Phase 0.
2. Xin/tải world thật; chạy doctor và baseline.
3. Ưu tiên `Gateway.decide` + guardrail vì enforcement, authority, safety và grounded answer có trọng số cao.
4. Hoàn thiện deck và prosecutor song song theo các checkpoint, giữ deck sẵn sàng khóa trước phiên.
5. Tích hợp, chạy toàn bộ kiểm tra cuối, nộp bundle đúng `TEAM`.

> Ghi chú: kế hoạch được xây từ nội dung hiện có trong `README.md`, `RULES.md` và tài liệu/module của kit. `CONTRACTS.md` được đề bài nói rõ là đặc tả giảng viên không phát cho sinh viên; không cần tìm hoặc phụ thuộc vào file này.

## Trạng thái thực hiện

- **Phase 0:** Python 3.12 đã được xác nhận, key gate đạt và test độc lập chạy được; world thật chưa có nên readiness/doctor đầy đủ đang chờ world export. Xem `mydoc/checkpoint1-baseline.md` phần môi trường.
- **Phase 1:** baseline fixture/demo đã được ghi; checkpoint spar với Rookie/Operator/Adversary chưa hoàn tất do thiếu world thật. Xem [checkpoint1-baseline.md](checkpoint1-baseline.md) để biết kết quả đo, lỗi demo starter và lệnh tái lập.
- **Phase 2:** gateway và answer-review API đã được triển khai trong `agent/`; demos và bộ kiểm thử tập trung đã đạt ở lần kiểm tra gần nhất. Chưa thể coi C2 tích hợp hoàn toàn: harness xử lý `ANSWER` ngoài `Gateway.decide`, và wiring tại `kit/loop` nằm ngoài thư mục được phép sửa. Xem [checkpoint2-defense.md](checkpoint2-defense.md) để biết phạm vi, kết quả và phần cần nối khi có answer-assembly hook.
