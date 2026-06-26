# Case 1 - Support Ticket Triage

## Mục tiêu

Case này giúp học viên luyện 4 câu hỏi nên bật ra ngay khi gặp một AI task:

- Cái gì deterministic và nên chấm bằng code?
- Cái gì cần semantic judgment và nên giao cho LLM judge hoặc human?
- Cái gì high-risk nên cần gate chặt hơn?
- Sai ở đâu thì cần escalation sang người thật?

Chỉ cần thiết kế eval ban đầu, không cần code full system.

Case này nối trực tiếp từ track **AI Customer Support Agent** ở Day 18/19, nhưng đổi góc nhìn từ **thiết kế trải nghiệm** sang **thiết kế eval**.

---

## 1. Bối cảnh

Một công ty SaaS B2B dùng AI để đọc ticket support mới và tạo output triage cho hệ thống nội bộ.

Output này không gửi trực tiếp cho khách hàng, nhưng nó được dùng để:

- phân loại ticket,
- đánh dấu mức độ gấp,
- route đến đúng team,
- quyết định có cần người thật nhảy vào hay không.

Nếu AI route sai, ticket có thể bị trễ, bỏ sót escalation, hoặc đẩy sai sang team không xử lý được.

---

## 2. Workflow logic (ASCII)

```text
Khách hàng gửi ticket hỗ trợ
    ↓
AI đọc:
- tiêu đề
- nội dung ticket
- loại khách hàng
    ↓
Hệ thống phải quyết định:
- đây là loại vấn đề gì?
- mức độ khẩn cấp ra sao?
- có cần người thật xử lý ngay không?
- ticket nên vào hàng của team nào?
    ↓
UI inbox nội bộ hiển thị:
- nhãn loại yêu cầu
- mức độ khẩn
- team phụ trách
- cờ "cần xử lý ngay"
- lý do tóm tắt
    ↓
Nếu khách doanh nghiệp + có dấu hiệu chặn công việc
    ↓
Đẩy lên hàng ưu tiên cao / escalation
```

---

## 3. UI hiển thị dự kiến (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Công ty ABC (Enterprise)                            |
| Tiêu đề: Thanh toán lỗi, tài khoản bị khóa                      |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: [ ? ]                                           |
| - Mức độ khẩn: [ ? ]                                            |
| - Team phụ trách: [ ? ]                                         |
| - Cần người xử lý ngay: [ ? ]                                   |
| - Lý do tóm tắt: [ .......................................... ] |
|----------------------------------------------------------------|
| Hàng đợi hiện tại: [ Bình thường ] hoặc [ Ưu tiên cao ]         |
+----------------------------------------------------------------+
```

Học viên cần tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Input mẫu

```json
{
  "ticket_id": "T-001",
  "subject": "Cannot login after password reset",
  "message": "I reset my password twice but still cannot log in. This is blocking my work.",
  "customer_tier": "enterprise"
}
```

Một input khác:

```json
{
  "ticket_id": "T-002",
  "subject": "URGENT: payment failed and account disabled",
  "message": "Our team is locked out because your billing system failed. Fix this now.",
  "customer_tier": "enterprise"
}
```

---

## 5. Business rules / operational rules

- Output phải đúng schema và đúng allowed enums.
- `confidence` phải nằm trong khoảng `0-1`.
- Nếu `customer_tier = enterprise` và `urgency` là `high` hoặc `critical`, `requires_human` phải bằng `true`.
- Ticket billing không được route sang `product_team`.
- Ticket có dấu hiệu “blocking work”, “locked out”, hoặc “account disabled” không nên bị đánh `low`.
- `reason_codes` phải phản ánh được nội dung ticket, không được bốc thêm sự thật không có trong input.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách hàng doanh nghiệp nhắn vào kênh hỗ trợ:

```text
Chị ơi bên em reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào được.
Bên em đang bị chặn công việc từ sáng.
```

### Data mẫu

- `customer_tier`: `enterprise`
- `account_name`: `Công ty Minh Phát Logistics`
- `previous_tickets_7d`: `0`
- `channel`: `Zalo OA`

### Workflow ASCII

```text
Khách nhắn vấn đề đăng nhập
    ↓
AI đọc nội dung + loại khách hàng
    ↓
AI phát hiện tín hiệu:
- login issue
- blocked work
- enterprise customer
    ↓
Hệ thống gợi ý:
- category = technical
- urgency = high hoặc critical
- requires_human = true
- route_to = technical_support
    ↓
UI nội bộ đẩy ticket lên hàng ưu tiên
    ↓
Nhân viên hỗ trợ xem lại rồi tiếp nhận
```

### UI trước khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Kênh: Zalo OA                                                   |
| Khách hàng: Minh Phát Logistics                                 |
|----------------------------------------------------------------|
| Nội dung khách nhắn:                                            |
| "Reset mật khẩu 2 lần rồi mà tài khoản admin vẫn không vào..."  |
|----------------------------------------------------------------|
| AI gợi ý: Chưa có                                               |
+----------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+----------------------------------------------------------------+
| Hộp thư hỗ trợ nội bộ                                           |
+----------------------------------------------------------------+
| Ticket: T-115                                                   |
| Khách hàng: Minh Phát Logistics (Enterprise)                    |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Technical                                       |
| - Mức độ khẩn: High                                             |
| - Team phụ trách: Technical Support                             |
| - Cần người xử lý ngay: Có                                      |
| - Lý do tóm tắt: Lỗi đăng nhập đang chặn công việc              |
| - Hàng đợi: Ưu tiên cao                                         |
+----------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang quyết định gì,
- quyết định nào hiển thị ra UI,
- và sai ở đâu thì ảnh hưởng vận hành.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là 3 seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Happy path

- `subject`: `Cannot login after password reset`
- Kỳ vọng: `category = technical`, `requires_human = true` nếu urgency đủ cao, route về `technical_support`

### Seed B - Ambiguous / low-info

- `subject`: `Help`
- `message`: `Please help asap`
- Kỳ vọng: AI không nên tự tin gán category quá mạnh; cần `unknown` hoặc route theo hướng cần review

### Seed C - High-risk / escalation

- `subject`: `URGENT: payment failed and account disabled`
- Kỳ vọng: `category = billing`, `urgency = critical`, `requires_human = true`, route về `billing_ops` hoặc `human_escalation`

---

## 8. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc seed cases ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, escalation, và regression.

1. **Happy path:** Ticket `enterprise` báo lỗi đăng nhập đang chặn việc → kỳ vọng `category=technical, urgency=high, requires_human=true, route_to=technical_support`.
   - Bắt failure: hệ thống có gán đúng và escalate đúng ở case "sách giáo khoa" không — nếu fail ở đây thì pipeline hỏng nặng.
2. **Ambiguous input:** Subject "Help", message "Please help asap", `tier=free` → kỳ vọng `category=unknown` hoặc confidence thấp + đẩy review.
   - Bắt failure: AI có overconfident gán category mạnh khi thiếu tín hiệu không.
3. **Missing information:** Ticket chỉ có subject, message rỗng, không có tín hiệu nội dung.
   - Bắt failure: AI có bịa lý do/route khi gần như không có dữ liệu, hay biết yêu cầu thêm thông tin.
4. **High-risk / escalation:** "URGENT: payment failed and account disabled", `tier=enterprise` → kỳ vọng `category=billing, urgency=critical, requires_human=true, route_to=billing_ops`.
   - Bắt failure: chính là lỗi mock T-002 — đánh nhẹ thành "Product question / Medium / không cần người". Bắt việc bỏ sót escalation high-stakes.
5. **Regression case:** Ticket billing nhưng diễn đạt như câu hỏi tính năng ("Sao gói Pro của tôi tự bị hủy giữa kỳ?") → kỳ vọng vẫn `category=billing`, không route `product_team`.
   - Bắt failure: rule "billing ⇏ product_team" và phân loại đúng khi ngôn ngữ gây nhiễu; pin lại để version sau không tái phạm.

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì? *(đã ghi ngay dưới mỗi case ở trên)*

---

## 9. Mock outcome để soi

Giả sử trên UI nội bộ, hệ thống hiển thị kết quả gợi ý như sau cho `T-002`:

```text
+----------------------------------------------------------------+
| Ticket: T-002                                                   |
| Khách hàng: Enterprise                                          |
|----------------------------------------------------------------|
| AI gợi ý                                                        |
| - Loại yêu cầu: Product question                                |
| - Mức độ khẩn: Medium                                           |
| - Team phụ trách: Support L1                                    |
| - Cần người xử lý ngay: Không                                   |
| - Lý do tóm tắt: Có vấn đề thanh toán                           |
| - Độ tin cậy: 0.91                                              |
+----------------------------------------------------------------+
```

Kết quả này trông có vẻ “ổn” nếu chỉ nhìn bề mặt, nhưng khả năng cao là sai về judgment vận hành.

---

## 10. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết eval runner,
- viết prompt judge thật,
- làm lại `User Input Grid` đầy đủ như bài test inputs hôm trước,
- tạo full dataset lớn,
- code full system.

Cần làm:

- chọn đúng nguồn chấm cho từng thành phần,
- viết các rule kiểm tra đủ cụ thể để có thể implement sau,
- đặt release gate có ý nghĩa vận hành,
- đề xuất 5 edge cases cần đưa vào reference dataset,
- và lập một pilot plan có thời gian + chi phí sơ bộ.

---

## 11. Bạn nên làm gì ở case 1?

Đây là case scaffold cao, nên cách làm tốt nhất là:

1. Đọc ví dụ full luồng trước để hiểu “một output tốt trông như thế nào”.
2. So mock outcome với ví dụ full luồng để thấy lỗi đang nằm ở đâu.
3. Nhìn từ UI để suy ra các field tối thiểu hệ thống phải có.
4. Điền `Eval Decision Map` trước, rồi mới quay lại viết các kiểm tra tự động và gate.

Case này thường **không bắt buộc phải có domain expert chuyên sâu**. Nếu chọn không cần expert, bạn vẫn phải giải thích vì sao human review vận hành là đủ.

---

## 12. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ hệ thống hỗ trợ khách hàng”.
- Ở case này, một `Unit of Work` tốt thường là: **một ticket đi vào -> AI gán nhãn, đánh mức ưu tiên, đề xuất route và cờ escalation**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval.

> **Lát cắt chọn:** Một ticket support mới đi vào (subject + message + customer_tier) → AI sinh ra một bản triage có cấu trúc gồm `category`, `urgency`, `route_to`, `requires_human`, `reason_summary` và `confidence`.
>
> Output này được dùng bởi **đội vận hành support nội bộ** (agent L1/L2, dispatcher) để đẩy ticket vào đúng hàng đợi và quyết định có escalate hay không. AI không trả lời khách trực tiếp.
>
> Đây là đơn vị đủ nhỏ vì chỉ xét **một ticket → một quyết định triage**: input rõ ràng, output có schema cố định nên dễ gán golden output và chấm lại. Nó vẫn chạm đúng rủi ro vận hành: nếu sai thì ticket bị route nhầm team, bỏ sót escalation cho khách enterprise đang bị chặn việc, hoặc ngược lại làm phồng hàng ưu tiên — đều là failure đo được trên đúng một case.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “AI có triage tốt không?”
- Nếu AI làm sai ở đây, điều gì sẽ khiến khách hàng mất trust hoặc không hoàn thành mục tiêu?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **AI có gắn đúng route và escalation để ticket không bị đi sai hàng xử lý không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì ticket sẽ đi sai hoặc gây mất trust.

> **Câu hỏi chất lượng:** "Với một ticket đi vào, AI có gán đúng `category`, đúng mức `urgency`, đúng `route_to`, và bật `requires_human` đúng lúc — đặc biệt với khách enterprise đang bị chặn việc — để ticket không bị đi sai hàng xử lý và không bị bỏ sót escalation không?"
>
> - **Behavior bắt buộc:** ticket billing phải về `billing_ops` (không về `product_team`); ticket có dấu hiệu "blocking work / locked out / account disabled" không được đánh `low`; enterprise + urgency high/critical phải `requires_human = true`.
> - **Behavior bị cấm:** bịa thêm sự thật không có trong ticket vào `reason_summary`; tự tin gán category khi input quá ít thông tin.
>
> Nếu fail ở đây: ticket khẩn của khách lớn nằm im trong hàng "bình thường" → khách bị chặn việc lâu, mất trust và có thể churn; hoặc route nhầm team khiến ticket bị đá qua lại, SLA vỡ. Đây là loại lỗi "trông ổn bề mặt" như mock outcome T-002 (billing bị gán Product question, Medium, confidence 0.91) nên rất nguy hiểm.

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- route đúng hàng xử lý,
- trigger escalation nếu cần,
- và chạy eval sau này.

Mẹo lấy từ ví dụ full luồng:

- Hãy nhìn ngược từ UI và mock outcome.
- Field nào không làm thay đổi màn hình, routing hoặc gate thì chưa cần đưa vào.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho UI, routing, escalation, hoặc eval.

> Output contract tối thiểu (kèm lý do giữ lại):
>
> - `ticket_id` (string) — khóa để nối input ↔ output ↔ golden khi chạy eval và để truy vết trên UI.
> - `category` (enum: `technical | billing | product | account | other | unknown`) — render nhãn "Loại yêu cầu" và là input cho rule routing; cần `unknown` cho case thiếu info (Seed B).
> - `urgency` (enum: `low | medium | high | critical`) — render "Mức độ khẩn", quyết định ticket vào hàng thường hay ưu tiên cao.
> - `route_to` (enum team: `technical_support | billing_ops | product_team | human_escalation | ...`) — quyết định ticket vào hàng của team nào; là field gây hậu quả vận hành lớn nhất nếu sai.
> - `requires_human` (bool) — bật cờ "cần xử lý ngay" và trigger escalation; là field safety quan trọng nhất.
> - `reason_summary` (string ngắn) — render "Lý do tóm tắt" cho agent hiểu nhanh và là phần để LLM judge chấm tính grounded.
> - `reason_codes` (array enum, vd `["blocking_work","enterprise","login_issue"]`) — tín hiệu có cấu trúc giải thích vì sao ra quyết định; dễ assert bằng code (vd có `blocking_work` thì không được `low`).
> - `confidence` (float 0–1) — dùng cho gate "confidence thấp → đẩy human review"; phải nằm trong [0,1].
>
> Bỏ qua những thứ không đổi UI/routing/gate (vd phân tích sentiment chi tiết, gợi ý câu trả lời cho khách) vì lát cắt này chỉ lo triage, chưa lo reply.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng chép lại toàn bộ business rules hay toàn bộ UI. Hãy chọn ra những thành phần quan trọng nhất, bám vào:

- `Output Contract` bạn đã đề xuất
- quyết định nào thật sự làm thay đổi route, escalation, hoặc safety
- chỗ nào nếu sai sẽ gây hậu quả vận hành rõ ràng

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Schema + enum hợp lệ, `confidence ∈ [0,1]` | ✅ | | | | Đúng/sai tuyệt đối, kiểm bằng JSON schema validator; không cần hiểu nghĩa. |
| Rule cứng: enterprise + high/critical ⇒ `requires_human=true`; billing ⇏ `product_team`; có `blocking_work` ⇏ `urgency=low` | ✅ | | | | Là invariant business rule, map thẳng input→điều kiện→output, deterministic nên code chặn chắc và rẻ. |
| `category` đúng so với nội dung ticket | | ✅ | (spot-check) | | Cần đọc hiểu ngữ nghĩa ticket; nhiều ticket mơ hồ nên code không phân loại tốt, LLM judge so với golden hợp lý hơn. |
| `urgency` hợp lý với mức độ nghiêm trọng mô tả | | ✅ | (spot-check) | | "Mức khẩn" là phán đoán mức độ, cần hiểu sắc thái ("bị chặn từ sáng" vs "khi nào tiện"); code chỉ bắt được keyword thô. |
| `reason_summary` có grounded, không bịa thêm fact | | ✅ | | | Hallucination check cần đối chiếu nghĩa giữa summary và message — đúng việc của LLM-as-judge. |
| Calibration bộ LLM judge + duyệt case high-risk/mơ hồ | | | ✅ | | Người vận hành dán nhãn golden, hiệu chỉnh judge, và review case escalation để bắt lỗi judgment nặng. |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nêu được vì sao bạn chọn nguồn chấm đó.

*Ghi chú: case này không cần cột Expert vì triage support không đòi chuyên môn ngành đặc thù; human review từ team vận hành là đủ (xem phần 7).*

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì ticket sẽ đi sai hàng, thiếu escalation, hoặc vỡ schema.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

- Kiểm tra: Output parse được thành JSON và đúng schema đã định nghĩa.
  Vì sao nên giao cho code: schema hợp lệ hay không là đúng/sai tuyệt đối; nếu vỡ schema thì UI không render được và toàn bộ eval phía sau vô nghĩa.
- Kiểm tra: `category`, `urgency`, `route_to` chỉ nhận giá trị trong tập enum cho phép.
  Vì sao nên giao cho code: giá trị lạ (vd `urgency = "very high"`) làm routing engine không khớp; so sánh với danh sách enum là việc rule thuần.
- Kiểm tra: `confidence` là số và nằm trong `[0, 1]`.
  Vì sao nên giao cho code: ràng buộc numeric, gate "confidence thấp" phụ thuộc vào nó nên phải đảm bảo miền giá trị.
- Kiểm tra: nếu `customer_tier = enterprise` và `urgency ∈ {high, critical}` thì `requires_human = true`.
  Vì sao nên giao cho code: là business rule cứng có điều kiện rõ ràng; bỏ sót sẽ làm vỡ SLA escalation — phải chặn chắc 100%, không để LLM "tùy hứng".
- Kiểm tra: nếu `category = billing` thì `route_to ≠ product_team`.
  Vì sao nên giao cho code: ràng buộc routing tuyệt đối, dễ verify bằng một phép so sánh.
- Kiểm tra: nếu message/`reason_codes` chứa tín hiệu `blocking_work | locked_out | account_disabled` thì `urgency ≠ low`.
  Vì sao nên giao cho code: là sàn an toàn dựa trên keyword/flag đã chuẩn hóa; chặn đúng loại lỗi "đánh nhẹ ticket nghiêm trọng".
- Kiểm tra: mọi phần tử trong `reason_codes` thuộc tập code hợp lệ đã định nghĩa.
  Vì sao nên giao cho code: tránh code rác làm hỏng phân tích coverage; so với enum là deterministic.
- Kiểm tra: `requires_human = true` thì `route_to` phải là một hàng có người (vd không route vào bot-only queue).
  Vì sao nên giao cho code: ràng buộc nhất quán nội bộ giữa hai field, verify bằng rule.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu nghĩa của ticket hoặc mức độ hợp lý của lý do tóm tắt.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

- Tiêu chí: `category` được gán có đúng với vấn đề thực sự mà khách mô tả không (so với golden label).
  Vì sao code không bắt tốt: phân loại intent cần đọc hiểu nội dung tự nhiên, nhiều ticket mơ hồ/đa ý; code chỉ match keyword nên dễ nhầm (vd "payment failed and account disabled" vừa billing vừa account).
- Tiêu chí: `urgency` có tương xứng với mức độ nghiêm trọng/tác động thực tế được mô tả không.
  Vì sao code không bắt tốt: mức khẩn là phán đoán sắc thái ("đang chặn việc cả team từ sáng" nặng hơn "lúc nào rảnh sửa giúp"); keyword không đo được cường độ.
- Tiêu chí: `reason_summary` có grounded vào message, không thêm fact không tồn tại trong input.
  Vì sao code không bắt tốt: phát hiện hallucination cần đối chiếu ngữ nghĩa giữa summary và ticket gốc; code không kiểm được "câu này có bịa không".
- Tiêu chí: với ticket low-info (vd "Help / Please help asap"), AI có thể hiện đúng sự không chắc chắn (dùng `unknown`/confidence thấp) thay vì gán bừa không.
  Vì sao code không bắt tốt: "có nên tự tin hay không" là đánh giá tính hợp lý của hành vi dưới điều kiện thiếu thông tin, cần judgment chứ không phải rule.
- Tiêu chí: `route_to` có phải lựa chọn hợp lý nhất trong các team khả dĩ không (ngoài các rule cứng đã có).
  Vì sao code không bắt tốt: khi không có rule tuyệt đối, chọn team "phù hợp nhất" cần hiểu bản chất vấn đề, là so sánh ngữ nghĩa.

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm người đó cần xem, và failure nào cần họ xem.

> **Ai review:** Support Operations Lead / Senior Agent (người trực tiếp vận hành hàng đợi triage).
>
> **Review những case nào:**
> - Case `requires_human = true` hoặc đẩy lên hàng ưu tiên cao (kiểm tra escalation có đúng không, vì đây là chỗ sai gây hậu quả nặng nhất).
> - Case `confidence` thấp hoặc `category = unknown` (AI tự báo không chắc).
> - Một mẫu ngẫu nhiên ticket bình thường để dò regression và hiệu chỉnh (calibrate) LLM judge định kỳ.
>
> **Vì sao đúng nhóm này:** họ hiểu taxonomy team, SLA và "đúng team" trong thực tế vận hành — thứ mà người ngoài hoặc LLM không nắm. Họ là người dán nhãn golden output và sửa rubric cho judge. Failure cần họ xem: escalation sai (bỏ sót hoặc thừa), route nhầm team gây ticket đá qua lại, và judgment "đánh nhẹ" ticket nghiêm trọng.
>
> **Có cần domain expert không? Không.** Triage support B2B SaaS không đòi chuyên môn ngành đặc thù (y khoa, pháp lý...); "đúng/sai" ở đây là kiến thức vận hành nội bộ mà Support Ops Lead đã có. Vì vậy human review vận hành là đủ; thêm expert chuyên sâu chỉ làm chậm và tốn chi phí không tương xứng.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review cho expert.

Expert cần thấy tối thiểu:

- AI đã route hoặc gắn nhãn gì,
- dấu hiệu hoặc evidence nào khiến case bị đẩy sang expert,
- expert có thể duyệt / sửa / escalation ở đâu.

**Trả lời của bạn:**

```text
Không áp dụng.
```

Không áp dụng — case triage support nội bộ chỉ cần human review từ Support Ops, không có quyết định đòi chuyên môn ngành đặc thù nên không lập màn hình/tiêu chí cho domain expert riêng.

#### 7B. Tiêu chí review của Domain Expert

Liệt kê các tiêu chí domain expert sẽ dùng để duyệt case này.

Không áp dụng (xem giải thích ở phần 7).

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Release Gate đề xuất** (chạy trên reference dataset trước khi ship một version prompt/model mới):

*Điều kiện chặn cứng (bất kỳ cái nào fail ⇒ block ship, không thương lượng):*
- Tỉ lệ pass các **code check** (schema, enum, confidence range, các business rule cứng) = **100%**. Đây là invariant; chỉ cần 1 case vi phạm rule "enterprise+critical ⇒ requires_human" là chặn.
- **0 case** route ticket billing sang `product_team` và **0 case** đánh `low` cho ticket có tín hiệu blocking.

*Ngưỡng chất lượng tối thiểu (đo bằng LLM judge đã calibrate + golden):*
- `category` accuracy ≥ **90%** trên reference dataset.
- `urgency` đúng hoặc lệch tối đa 1 bậc ≥ **90%**; **0 case** đánh nhẹ ≥ 2 bậc với ticket high-risk.
- **Recall của `requires_human` trên nhóm case cần escalation = 100%** (thà thừa escalation còn hơn bỏ sót).
- Tỉ lệ `reason_summary` bị đánh "có hallucination" ≤ **5%**.

*Trường hợp bắt buộc human review (không tự ship dù qua gate số):*
- Mọi case `requires_human = true` ở production được Support Ops kiểm lại trong giai đoạn pilot.
- LLM judge bị phát hiện lệch chuẩn (precision/recall so với human < 0.8) ⇒ dừng dùng judge, quay lại human label trước khi tin số liệu.
- Có regression so với version trước trên bất kỳ case escalation high-risk nào.

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài triage này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- cần thêm những checkpoint nào trước khi đề xuất triển khai tiếp
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ vận hành / kỹ thuật
- tổng giờ human review
- nếu có `domain expert`, tổng số giờ expert
- tổng chi phí API key
- tổng chi phí pilot
- tổng thời gian dự kiến

Có thể lấy mốc tham khảo để nhẩm nhanh:

- khoảng `50-100 cases`
- khoảng `30-50 lần chạy / lặp lại`

Không cần trình bày thành bảng. Hãy tự chọn cách trình bày miễn là người đọc nhìn vào hiểu được bạn đã tính gì và chi phí tổng rơi vào đâu.

Sau phần này, viết thêm 2-4 câu ngắn:

- bạn dùng giá API thật từ đâu để tính,
- với quy mô này chi phí tổng rơi vào khoảng nào,
- và vì sao plan này đủ để chứng minh case có thể pilot được.

**Kế hoạch chạy thử và dự toán (Case 1 — Ticket Triage):**

*Giả định quy mô:*
- Reference dataset pilot: **80 cases** (3 seed + 5 edge case mình đề xuất, mở rộng theo các lát cắt).
- Số lần chạy/lặp: **40 lần** (lặp lại khi sửa prompt/model, chạy lại regression) ⇒ **80 × 40 = 3.200 lượt gọi triage**.
- Mỗi lượt triage: ~**1.500 token input** (subject + message + system prompt + schema) + ~**400 token output** (JSON triage). LLM judge chạy thêm 1 lượt/lượt triage: ~**1.200 token input** + ~**150 token output**.

*Giá API thật dùng để tính (giá công khai Anthropic, tham chiếu qua skill `claude-api`):*
- **Claude Haiku 4.5** cho model triage: **$1 / 1M input, $5 / 1M output**.
- **Claude Sonnet 4.6** cho LLM judge (cần hiểu ngữ nghĩa tốt hơn): **$3 / 1M input, $15 / 1M output**.

*Chi phí API:*
- Triage (Haiku): input 3.200 × 1.500 = 4,8M token → 4,8 × $1 = **$4,8**; output 3.200 × 400 = 1,28M token → 1,28 × $5 = **$6,4**. ⇒ ~**$11,2**.
- Judge (Sonnet): input 3.200 × 1.200 = 3,84M token → 3,84 × $3 = **$11,5**; output 3.200 × 150 = 0,48M token → 0,48 × $15 = **$7,2**. ⇒ ~**$18,7**.
- **Tổng API ≈ $30** (làm tròn lên ~$35–40 để dự phòng prompt dài/lặp thêm).

*Giờ công (người):*
- PM / thiết kế eval: **20 giờ** (viết quality question, output contract, decision map, edge cases, release gate).
- Kỹ thuật / vận hành (viết code checks + dựng runner đơn giản + chạy): **24 giờ**.
- Human review (Support Ops dán nhãn golden 80 cases + calibrate judge + review case escalation): **16 giờ**.
- Domain expert: **0 giờ** (không cần — đã giải thích ở phần 7).
- Tổng giờ công ≈ **60 giờ**.

*Tổng kết:*
- **Chi phí API ≈ $35–40.** Phần tiền mặt thực sự rất nhỏ; chi phí chính là **~60 giờ công người**.
- **Thời gian dự kiến: ~1,5–2 tuần** (1 tuần thiết kế + dán nhãn, vài ngày chạy + hiệu chỉnh).

Mình dùng giá API công khai của Anthropic (Haiku 4.5 và Sonnet 4.6) làm mốc tính. Với quy mô ~80 cases × 40 lần chạy, chi phí máy chỉ vài chục đô, nên budget xin chủ yếu là giờ công (~60 giờ). Plan này đủ để chứng minh case pilot được vì nó cho ra **con số accuracy thực trên dataset có golden**, đo được recall escalation (chỉ số rủi ro lớn nhất), và xác định liệu LLM judge có đáng tin (đối chiếu với human) trước khi đề xuất scale.

---
