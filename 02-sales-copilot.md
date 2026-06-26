# Case 2 - Sales Chat Copilot: Tóm tắt hội thoại và tra cứu theo tín hiệu khách gửi

## Mục tiêu

Case này đại diện cho một kiểu AI rất hay gặp ở thị trường Việt Nam:

- đội sales hoặc chăm sóc khách hàng đang nhắn tin với khách qua Zalo, Facebook, live chat hoặc CRM inbox,
- AI không thay người chốt đơn hoàn toàn,
- nhưng AI có thể đọc hội thoại, tóm tắt ngữ cảnh, phát hiện tín hiệu như số điện thoại / email / mã đơn / mã khách,
- rồi tra cứu hệ thống nội bộ để đưa thông tin cần thiết cho nhân viên xử lý nhanh hơn.

Điểm khó của case này là:

- có nhiều logic lookup tự động,
- có nguy cơ match sai người hoặc sai đơn,
- có dữ liệu nhạy cảm,
- và rất dễ “trông thông minh” dù thực tế đang suy luận sai.

Chỉ cần thiết kế eval ban đầu, không cần code full system.

---

## 1. Bối cảnh

Một doanh nghiệp bán hàng online tại Việt Nam có đội sales / chăm sóc khách hàng xử lý tin nhắn từ nhiều kênh:

- Zalo OA,
- Facebook Messenger,
- web chat,
- CRM inbox nội bộ.

Khi khách nhắn tin, nhân viên thường phải làm nhiều việc thủ công:

- đọc lại lịch sử hội thoại,
- hiểu khách đang hỏi về vấn đề gì,
- tự tìm số điện thoại / email / mã đơn trong đoạn chat,
- tra CRM hoặc hệ thống đơn hàng,
- rồi mới biết khách này là ai, đang ở trạng thái nào, đơn hàng nào liên quan và nên trả lời tiếp ra sao.

Nhóm muốn thêm một **Sales Chat Copilot** để:

- tóm tắt cuộc hội thoại hiện tại,
- phát hiện các mã hoặc thông tin nhận diện khách,
- tra cứu nhanh hồ sơ khách / đơn hàng / ticket cũ,
- gợi ý thông tin quan trọng cho nhân viên,
- và có thể gợi ý câu trả lời nháp.

AI **không được tự gửi tin nhắn**, **không được tự chốt đơn**, và **không được tự sửa dữ liệu khách**.

---

## 2. Workflow logic tham khảo (ASCII)

```text
Khách nhắn tin đến từ Zalo / Facebook / web chat
    ↓
AI đọc:
- tin nhắn mới nhất
- lịch sử hội thoại gần đây
- metadata kênh nhắn tin
    ↓
AI phát hiện tín hiệu:
- số điện thoại
- email
- mã đơn
- mã khách
- tên sản phẩm / nhu cầu mua
    ↓
Nếu có tín hiệu đủ mạnh
    ↓
Tra CRM / OMS / ticket history
    ↓
Hiển thị cho nhân viên:
- tóm tắt hội thoại
- khách đang hỏi gì
- hồ sơ / đơn hàng liên quan
- cảnh báo nếu có điểm mâu thuẫn
- gợi ý bước tiếp theo
    ↓
Nhân viên xem lại và quyết định:
- trả lời thủ công
- dùng nháp AI rồi sửa
- hỏi thêm khách
- chuyển sale khác / chuyển CSKH / escalate
```

---

## 3. Khung UI gợi ý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Khách: Chưa xác định chắc chắn                      |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456                                  |
| Khách: Mã đơn hình như là DH-48291                                                             |
| NV sale: ...                                                                                   |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: [ ................................................................. ]    |
| - Tín hiệu phát hiện: [ số điện thoại ] [ mã đơn ]                                            |
| - Khách / đơn liên quan: [ ? ]                                                                 |
| - Cảnh báo: [ Có / Không ]                                                                     |
| - Gợi ý bước tiếp theo: [ ............................................................... ]    |
|-----------------------------------------------------------------------------------------------|
| [Xem hồ sơ] [Xem đơn hàng] [Dùng nháp AI] [Hỏi thêm khách] [Chuyển người xử lý]              |
+------------------------------------------------------------------------------------------------+
```

Học viên có thể dùng khung này hoặc chỉnh lại nếu thấy logic sản phẩm cần thay đổi. Điểm quan trọng là phải tự đề xuất output contract tối thiểu phía sau để màn hình này hiển thị được.

---

## 4. Tình huống mẫu

### Tình huống A - Khách gửi số điện thoại

```text
Khách: Chị check giúp em đơn cũ với, số em là 0909123456.
```

### Tình huống B - Khách gửi email

```text
Khách: Bên mình kiểm tra lại giúp mình email linh.nguyen@abc.vn nhé, hôm trước có tư vấn mà chưa thấy báo giá.
```

### Tình huống C - Khách gửi mã đơn

```text
Khách: Em hỏi đơn DH-48291 đang tới đâu rồi ạ?
```

### Tình huống D - Khách hỏi mơ hồ

```text
Khách: Chị ơi bên em xử lý giúp case này với, gấp lắm.
```

---

## 5. Business rules / operational rules

- AI có thể **đề xuất tra cứu** hoặc **tự động tra cứu nội bộ** nếu tín hiệu nhận diện đủ rõ, nhưng không được tự gửi phản hồi cho khách.
- Nếu có nhiều bản ghi cùng khớp với một số điện thoại / email / mã, AI không được tự chốt một bản ghi duy nhất.
- Nếu không tìm thấy hồ sơ phù hợp, AI phải nói rõ là “chưa tìm thấy”, không được bịa ra khách hoặc đơn.
- Nếu khách gửi dữ liệu nhạy cảm, hệ thống chỉ hiển thị phần cần thiết cho nhân viên, không bung toàn bộ dữ liệu không liên quan.
- Nếu kết quả CRM và kết quả đơn hàng mâu thuẫn, AI phải cảnh báo mức độ không chắc chắn.
- AI có thể gợi ý nháp trả lời, nhưng bản nháp phải được nhân viên xem lại trước khi gửi.
- AI không được tự tạo đơn mới chỉ vì phát hiện nhu cầu mua, trừ khi luồng đó được người dùng nội bộ xác nhận rõ.

---

## 6. Ví dụ full luồng để hình dung nhanh

### Tình huống

Khách nhắn vào Zalo OA:

```text
Chị kiểm tra giúp em đơn DH-48291 với ạ.
Số em là 0909123456.
Hôm trước chị tư vấn cho em máy lọc nước rồi.
```

### Data mẫu

**Dữ liệu từ CRM**

- Tên khách: `Nguyễn Minh Linh`
- Số điện thoại: `0909123456`
- Kênh lead: `Zalo OA`
- Sales owner: `Trâm`
- Trạng thái lead: `Đã mua lần 1`

**Dữ liệu từ OMS**

- Mã đơn: `DH-48291`
- Sản phẩm: `Máy lọc nước RO Mini`
- Trạng thái: `Đang giao`
- Dự kiến giao: `Hôm nay`

**Lịch sử hội thoại gần đây**

- Khách từng hỏi báo giá
- Sales đã tư vấn 2 model
- Khách đã chốt 1 model hôm trước

### Workflow ASCII

```text
Khách gửi mã đơn + số điện thoại
    ↓
AI phát hiện 2 tín hiệu tra cứu:
- mã đơn
- số điện thoại
    ↓
AI tra CRM + OMS
    ↓
Hệ thống nối thông tin:
- đây là khách cũ
- đơn DH-48291 đang giao
- sales owner hiện tại là Trâm
    ↓
Copilot hiển thị:
- tóm tắt hội thoại
- hồ sơ khách
- trạng thái đơn
- gợi ý bước tiếp theo
    ↓
Nhân viên sales đọc lại
    ↓
Nhân viên chọn:
- dùng nháp AI rồi sửa
- xem đơn chi tiết
- hỏi thêm khách
```

### UI trước khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                                                                                  |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot: Chưa phân tích                                                                        |
+------------------------------------------------------------------------------------------------+
```

### UI sau khi AI xử lý (ASCII)

```text
+------------------------------------------------------------------------------------------------+
| Hộp chat khách hàng                                                                            |
+------------------------------------------------------------------------------------------------+
| Kênh: Zalo OA                               Sales owner hiện tại: Trâm                         |
|-----------------------------------------------------------------------------------------------|
| Khách: Chị kiểm tra giúp em đơn DH-48291 với ạ.                                               |
| Khách: Số em là 0909123456.                                                                    |
|-----------------------------------------------------------------------------------------------|
| Copilot                                                                                       |
| - Tóm tắt hội thoại: Khách cũ đang hỏi về tình trạng đơn vừa mua.                             |
| - Tín hiệu phát hiện: [0909123456] [DH-48291]                                                  |
| - Hồ sơ khách: Nguyễn Minh Linh - Đã mua lần 1                                                 |
| - Đơn hàng: Máy lọc nước RO Mini - Trạng thái: Đang giao                                       |
| - Gợi ý: Báo khách đơn đang giao hôm nay và xác nhận khung giờ nhận hàng.                     |
| - Nháp trả lời: "Dạ em thấy đơn DH-48291 đang được giao hôm nay..."                            |
|-----------------------------------------------------------------------------------------------|
| [Xem CRM] [Xem đơn] [Dùng nháp AI] [Hỏi thêm khách]                                           |
+------------------------------------------------------------------------------------------------+
```

Ví dụ này giúp người đọc hình dung ngay:

- AI đang tự động bắt tín hiệu gì,
- lookup nào đang diễn ra ở hậu trường,
- thông tin nào được kéo ra cho sales,
- và ranh giới giữa “gợi ý” với “tự hành động”.

---

## 7. Seed cases

Đây không phải full dataset. Đây chỉ là các seed cases để học viên hình dung phạm vi và failure modes.

### Seed A - Match rõ, một bản ghi duy nhất

- Khách gửi đúng số điện thoại.
- CRM trả về đúng một hồ sơ.
- Có một đơn hàng gần nhất đang giao.
- Kỳ vọng: AI tóm tắt đúng và hiển thị đúng đơn liên quan.

### Seed B - Một tín hiệu khớp nhiều bản ghi

- Cùng một số điện thoại gắn với hai hồ sơ do nhập trùng.
- Kỳ vọng: AI phải cảnh báo mơ hồ và yêu cầu nhân viên chọn đúng hồ sơ.

### Seed C - Khách gửi mã đơn hợp lệ nhưng không phải khách hiện tại

- Mã đơn tồn tại trong hệ thống nhưng thuộc người khác.
- Kỳ vọng: AI không được suy ra đây chắc chắn là đơn của người đang chat.

### Seed D - Khách hỏi mơ hồ, chưa có tín hiệu tra cứu

- Kỳ vọng: AI không nên bịa hồ sơ; nên gợi ý hỏi thêm số điện thoại / email / mã đơn.

### Seed E - Dữ liệu hệ thống mâu thuẫn

- CRM nói khách đang là lead mới.
- OMS lại có lịch sử đơn cũ.
- Kỳ vọng: AI phải nêu rõ điểm mâu thuẫn thay vì tóm tắt như thể mọi thứ đã chắc chắn.

---

## 8. Mock outcome để soi

Giả sử trên UI nội bộ, Copilot hiển thị như sau sau khi khách gửi:

```text
Khách: Chị kiểm tra giúp em đơn này với, em gửi số 0909123456. Mã đơn DH-48291.
```

```text
+------------------------------------------------------------------------------------------------+
| Copilot                                                                                        |
+------------------------------------------------------------------------------------------------+
| Tóm tắt hội thoại: Khách hỏi về tình trạng đơn hàng cũ.                                        |
| Tín hiệu phát hiện: 0909123456, DH-48291                                                       |
| Khách liên quan: Nguyễn Minh Linh                                                              |
| Đơn liên quan: DH-48291 - Đã giao thành công                                                   |
| Cảnh báo: Không                                                                                |
| Gợi ý cho sales: Báo khách đơn đã giao và mời mua thêm sản phẩm mới.                           |
+------------------------------------------------------------------------------------------------+
```

Kết quả này có thể trông “mượt”, nhưng vẫn có khả năng rất nguy hiểm nếu:

- số điện thoại khớp nhiều người,
- mã đơn thuộc khách khác,
- trạng thái “đã giao” là dữ liệu cũ,
- hoặc AI đang đẩy sales upsell sai thời điểm.

---

## 9. Bộ test gợi ý v0

Phần này chỉ để gợi ý cách nghĩ coverage từ bài hôm trước. Không phải deliverable bắt buộc.

Bạn có thể dùng 8 tình huống dưới đây để nghĩ coverage ban đầu:

| ID | Tình huống | Điều cần bắt |
| --- | --- | --- |
| SC-01 | Khách gửi số điện thoại đúng format, match 1 hồ sơ | lookup đúng |
| SC-02 | Khách gửi email viết hoa/thường lẫn lộn | normalization |
| SC-03 | Khách gửi mã đơn sai 1 ký tự | không tự match bừa |
| SC-04 | Cùng số điện thoại khớp 2 hồ sơ | ambiguity handling |
| SC-05 | Khách chỉ nói “chị kiểm tra giúp em case này” | ask for missing info |
| SC-06 | CRM và OMS mâu thuẫn | uncertainty + warning |
| SC-07 | AI gợi ý nháp trả lời nhưng nhầm intent bán hàng thành hậu mãi | summary / intent error |
| SC-08 | Khách gửi tiếng Việt không dấu + mã đơn | robustness với ngôn ngữ thực tế |

---

## 10. Bạn phải đề xuất thêm 5 Dataset Edge Cases

Sau khi đọc bộ test gợi ý v0 ở trên, hãy đề xuất thêm 5 case cần đưa vào reference dataset version đầu.

Không cần nộp một bảng coverage riêng. Hãy chọn 5 case đại diện cho các lát cắt khác nhau, ví dụ: match rõ, thiếu tín hiệu, ambiguity, dữ liệu mâu thuẫn, và action safety.

1. **Happy path:** Khách gửi đúng 1 SĐT khớp duy nhất 1 hồ sơ + 1 đơn gần nhất đang giao → kỳ vọng `single_match`, tóm tắt đúng, không cảnh báo.
   - Bắt failure: pipeline cơ bản (trích → lookup → summary) có chạy đúng ở case sạch không.
2. **Ambiguous lookup:** Một SĐT khớp 2 hồ sơ (do nhập trùng) → kỳ vọng `multiple_matches` + `warnings.ambiguous_match`, yêu cầu nhân viên chọn.
   - Bắt failure: AI có tự chốt 1 bản ghi khi mơ hồ không — rủi ro match nhầm người.
3. **Missing information:** Khách nhắn "chị xử lý giúp em case này, gấp lắm" (không có tín hiệu) → kỳ vọng `not_found`/không lookup, gợi ý hỏi thêm SĐT/email/mã.
   - Bắt failure: AI có bịa hồ sơ khi chưa có tín hiệu, hay biết dừng lại hỏi thêm.
4. **Conflicting systems:** CRM nói "lead mới" nhưng OMS có lịch sử đơn cũ → kỳ vọng `warnings.data_conflict`, nêu rõ mức không chắc chắn.
   - Bắt failure: AI có tóm tắt như thể mọi thứ đã chắc chắn, bỏ qua mâu thuẫn không.
5. **Regression case:** Mã đơn hợp lệ nhưng thuộc khách khác (không phải người đang chat) → kỳ vọng không suy ra đây chắc chắn là đơn của khách hiện tại.
   - Bắt failure: pin lại lỗi "match nhầm đơn sang người đang chat" để version sau không tái phạm.

Với mỗi case, thêm 1 dòng ngắn giải thích:

- case này dùng để bắt failure gì? *(đã ghi ngay dưới mỗi case ở trên)*

---

## 11. Nhiệm vụ học viên

Hãy điền workbook bên dưới cho case này.

Không cần:

- viết connector CRM thật,
- viết regex hoặc parser thật,
- làm lại `User Input Grid` hay `Scenario Dataset v0/v1`,
- code full workflow,
- dựng UI đẹp.

Cần làm:

- xác định unit of AI work đủ nhỏ,
- viết quality question cho lát cắt này,
- đề xuất output contract tối thiểu,
- quyết định phần nào chấm bằng code / LLM / human,
- đặt release gate cho hành vi tra cứu và hiển thị gợi ý,
- đề xuất edge cases cho dataset,
- và lập pilot plan có thời gian + chi phí sơ bộ.

---

## 12. Bạn nên làm gì ở case 2?

Đây là case scaffold trung bình, nên bạn không cần giữ nguyên workflow và UI gợi ý.

Nên làm theo thứ tự:

1. Dùng workflow tham khảo để xác định các bước chính của hệ thống.
2. Quyết định chỗ nào AI được tự lookup, chỗ nào phải hỏi thêm.
3. Xem lại khung UI và chỉnh nếu bạn thấy thiếu checkpoint quan trọng.
4. Chỉ sau đó mới chốt output contract và release gate.

Case này thường thiên về **ops / sales / CRM** hơn là domain chuyên môn sâu. Nếu chọn **không cần domain expert**, bạn vẫn phải giải thích rõ vì sao chỉ cần human review từ team vận hành là đủ.

---

## 13. Workbook

Lưu ý chung cho toàn bộ câu trả lời:

- Không chỉ điền đáp án ngắn.
- Với mỗi phần, hãy nêu cả **quyết định** và **lý do**.
- Nếu chỉ liệt kê mà không giải thích vì sao, bài sẽ khó được xem là hiểu thật.

### 1. Unit of Work

- AI đang thực hiện công việc gì?
- Output cuối cùng được dùng bởi ai?
- Nếu sai, hậu quả vận hành là gì?

Gợi ý từ bài hôm trước:

- Đừng chọn “toàn bộ Copilot cho sales”.
- Ở case này, một `Unit of Work` tốt thường là: **một tín hiệu khách gửi vào -> AI phát hiện tín hiệu -> tra cứu -> tóm tắt -> gợi ý bước tiếp theo**.

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- bạn chọn lát cắt nào,
- và vì sao đây là đơn vị đủ nhỏ để eval mà vẫn chạm đúng rủi ro vận hành.

> **Lát cắt chọn:** Một tin nhắn khách gửi vào (kèm lịch sử hội thoại gần đây) → Copilot phát hiện tín hiệu nhận diện (SĐT/email/mã đơn/mã khách), tra cứu CRM/OMS, rồi sinh ra một khối gợi ý gồm: tóm tắt hội thoại, hồ sơ/đơn liên quan, cảnh báo (ambiguity/mâu thuẫn), và gợi ý bước tiếp theo.
>
> Output này được **nhân viên sales/CSKH** đọc để trả lời khách nhanh hơn; AI **không tự gửi tin, không tự chốt đơn, không tự sửa data**.
>
> Đây là đơn vị đủ nhỏ vì xét đúng **một lượt khách nhắn → một khối gợi ý**: input rõ (tin nhắn + data CRM/OMS giả lập), output có cấu trúc nên gán golden được. Nó chạm đúng rủi ro vận hành nguy hiểm nhất: **match nhầm người/nhầm đơn** rồi tóm tắt như thể đã chắc chắn — khiến nhân viên tin và trả lời sai khách.

### 2. Quality Question

Viết một câu hỏi chất lượng đủ cụ thể cho lát cắt này.

Gợi ý:

- Đừng hỏi kiểu quá rộng như: “Copilot này có hữu ích không?”
- Nếu Copilot lookup sai hoặc summary sai, điều gì sẽ làm nhân viên mất trust hoặc trả lời sai khách?
- Behavior nào là bắt buộc?
- Behavior nào là bị cấm?
- Viết theo dạng: **Copilot có lookup đúng hồ sơ và biết dừng lại khi xuất hiện ambiguity hoặc mâu thuẫn không?**

**Trả lời của bạn:**

Hãy viết 2-4 câu, trong đó có cả:

- câu hỏi chất lượng bạn chọn,
- và vì sao nếu fail ở đây thì sales có thể mất trust hoặc trả lời sai khách.

> **Câu hỏi chất lượng:** "Với một tín hiệu khách gửi vào, Copilot có lookup đúng hồ sơ/đơn, và **biết dừng lại cảnh báo** khi tín hiệu mơ hồ (khớp nhiều bản ghi), không tìm thấy, hoặc dữ liệu CRM↔OMS mâu thuẫn — thay vì tự tin chốt một kết quả duy nhất không?"
>
> - **Behavior bắt buộc:** khi 1 SĐT/email/mã khớp nhiều bản ghi → cảnh báo ambiguity, yêu cầu nhân viên chọn; khi không tìm thấy → nói rõ "chưa tìm thấy"; khi CRM↔OMS mâu thuẫn → nêu rõ mức không chắc chắn.
> - **Behavior bị cấm:** bịa hồ sơ/đơn không tồn tại; tự chốt 1 bản ghi khi có nhiều khả năng; bung toàn bộ dữ liệu nhạy cảm không liên quan; tự gửi tin hoặc tự tạo đơn.
>
> Nếu fail ở đây: nhân viên tin vào một match sai (mã đơn thuộc khách khác, hoặc trạng thái "đã giao" là data cũ) rồi nhắn cho khách thông tin sai → mất trust, lộ nhầm thông tin người khác, hoặc upsell sai thời điểm. Đây chính là lỗi của mock outcome ở phần 8 (báo "đã giao" + mời mua thêm dù dữ liệu chưa chắc chắn).

### 3. Output Contract tối thiểu

Không cần đoán full JSON hoàn chỉnh. Chỉ cần đề xuất những field tối thiểu mà hệ thống phải có ở backend hoặc trace để:

- render UI ở trên,
- hiển thị summary và tín hiệu phát hiện,
- gắn đúng hồ sơ / đơn hàng liên quan,
- cảnh báo khi có ambiguity hoặc mâu thuẫn,
- và chạy eval sau này.

Mẹo:

- Hãy nhìn từ khung UI gợi ý để quyết định field nào thật sự cần hiện ra.
- Field nào chỉ “hay thì có” nhưng không ảnh hưởng lookup, cảnh báo hoặc next step thì chưa cần.

**Trả lời của bạn:**

Đừng chỉ liệt kê field. Với mỗi field bạn giữ lại, hãy giải thích ngắn vì sao nó cần cho lookup, summary, ambiguity warning, next step, hoặc eval.

> Output contract tối thiểu (kèm lý do):
>
> - `conversation_id` / `message_id` — khóa nối input ↔ output ↔ golden để chạy eval và truy vết.
> - `detected_signals` (array, mỗi phần tử `{type: phone|email|order_id|customer_id, value, normalized}`) — render khối "Tín hiệu phát hiện"; `normalized` cần để lookup nhất quán (email hoa/thường, SĐT có/không +84).
> - `lookup_results` (array `{source: CRM|OMS, record_id, matched_on, confidence}`) — quyết định hồ sơ/đơn nào được kéo ra; `matched_on` + nhiều phần tử là cơ sở để phát hiện ambiguity.
> - `match_status` (enum: `single_match | multiple_matches | not_found`) — field then chốt cho cảnh báo: chỉ `single_match` mới được hiển thị chắc chắn.
> - `warnings` (array enum: `ambiguous_match | data_conflict | stale_data | sensitive_data_hidden`) — render khối "Cảnh báo"; trigger nhân viên kiểm lại trước khi trả lời.
> - `summary` (string) — render "Tóm tắt hội thoại"; là phần LLM judge chấm tính grounded.
> - `suggested_next_step` (string + enum action gợi ý: `reply_draft | ask_more_info | transfer | none`) — render gợi ý; phân biệt "gợi ý" với "tự hành động".
> - `reply_draft` (string, nullable) — bản nháp; phải để nhân viên duyệt trước khi gửi (không tự gửi).
>
> Bỏ qua field "hay thì có" nhưng không đổi lookup/cảnh báo/next step (vd điểm sentiment chi tiết, toàn bộ lịch sử mua hàng) — chưa cần ở lát cắt này.

### 4. Eval Decision Map

Ở phần này, bạn phải **tự quyết định** đâu là các thành phần thật sự cần chấm.

Đừng biến bảng này thành đáp án chép lại từ đề. Hãy tự chọn các thành phần dựa trên:

- `Output Contract` bạn đã đề xuất
- workflow lookup / summary / suggestion mà bạn chọn
- chỗ nào nếu sai sẽ làm match nhầm, hiểu sai, hoặc act quá quyền hạn

| Thành phần cần chấm | Code | LLM | Human | Expert | Lý do |
| --- | ---: | ---: | ---: | ---: | --- |
| Trích & chuẩn hóa tín hiệu (SĐT/email/mã đúng format, normalize đúng) | ✅ | | | | Regex + chuẩn hóa là deterministic; so output với golden tín hiệu là đúng/sai rõ ràng. |
| `match_status` đúng so với data: nhiều bản ghi ⇒ `multiple_matches`, không có ⇒ `not_found` | ✅ | | | | Đếm số record khớp trong DB giả lập là phép logic; đây là sàn an toàn chống "tự chốt 1 bản ghi". |
| Có `warnings` đúng khi data CRM↔OMS mâu thuẫn / stale | ✅ | (spot-check) | | | Mâu thuẫn = so sánh field giữa 2 nguồn theo rule; code bắt được. Human spot-check để chắc rule phủ đúng. |
| Không lộ field nhạy cảm ngoài phần cần thiết | ✅ | | | | Kiểm field nào được render so với allowlist — ràng buộc cứng, verify bằng code. |
| `summary` có grounded, không bịa hồ sơ/đơn không có trong lookup | | ✅ | | | Cần đối chiếu nghĩa giữa summary và lookup_results; hallucination check là việc của LLM. |
| `suggested_next_step` có hợp lý & đúng ranh giới (không xui tự chốt/tự gửi/upsell sai lúc) | | ✅ | (review) | | Tính hợp lý của gợi ý cần đọc hiểu ngữ cảnh hội thoại; code không đánh giá được "đúng thời điểm". |
| Calibrate LLM judge + review case match nhầm / mâu thuẫn | | | ✅ | | Sales/CRM Ops dán nhãn golden, hiệu chỉnh judge, và soi case có rủi ro match sai người. |

Bạn có thể thêm hoặc bớt dòng nếu cần, nhưng không nên biến bảng này thành một danh sách rất dài.

Không chấp nhận bảng chỉ tick `Yes/No`. Cột `Lý do` phải nói rõ vì sao thành phần đó nên giao cho code, LLM, human, hay expert.

*Ghi chú: không cần cột Expert — đây là bài toán ops/sales/CRM, "đúng/sai" là kiến thức vận hành nội bộ; human review từ team Sales Ops là đủ (xem phần 7).*

### 5. Kiểm tra tự động bằng code

Liệt kê **đầy đủ** các rule kiểm tra tự động mà bạn cho rằng case này cần có.

Không giới hạn số lượng. Hãy coi như bạn đang thiết kế bộ eval thật cho chính bài toán này, không phải chỉ chọn vài ý tiêu biểu.

Ưu tiên các kiểm tra mà nếu fail thì Copilot sẽ match nhầm hồ sơ, match nhầm đơn, hoặc act vượt quyền.

Mỗi ý nên viết theo dạng:

- Kiểm tra: [rule]
  Vì sao nên giao cho code:

- Kiểm tra: Output parse được thành JSON đúng schema; các enum (`match_status`, `warnings`, `action`) đúng tập cho phép.
  Vì sao nên giao cho code: vỡ schema/enum lạ làm UI không render và routing/gate sai; so với schema là đúng/sai tuyệt đối.
- Kiểm tra: SĐT/email/mã đơn được trích đúng và `normalized` đúng quy tắc (SĐT về dạng chuẩn, email lowercase, mã đơn đúng pattern `DH-xxxxx`).
  Vì sao nên giao cho code: trích & chuẩn hóa là regex/string rule deterministic; so với golden tín hiệu rất rõ ràng.
- Kiểm tra: `match_status` khớp số bản ghi thực trong DB giả lập (nhiều record ⇒ `multiple_matches`; 0 record ⇒ `not_found`; đúng 1 ⇒ `single_match`).
  Vì sao nên giao cho code: chỉ là đếm số record khớp — chặn chắc lỗi "tự chốt 1 hồ sơ" mà không cần hiểu ngữ nghĩa.
- Kiểm tra: khi `match_status ≠ single_match` thì **không** được hiển thị một hồ sơ/đơn như đã xác định chắc chắn (phải có `warnings.ambiguous_match` hoặc `not_found`).
  Vì sao nên giao cho code: ràng buộc nhất quán giữa field; verify bằng một phép kiểm logic.
- Kiểm tra: khi field tương ứng ở CRM và OMS khác nhau (vd trạng thái lead vs lịch sử đơn) thì có `warnings.data_conflict`.
  Vì sao nên giao cho code: so sánh field giữa hai nguồn theo rule là deterministic.
- Kiểm tra: mã đơn được trích phải tồn tại trong OMS mới được gắn là đơn liên quan (mã sai 1 ký tự ⇒ không match bừa).
  Vì sao nên giao cho code: tra cứu tồn tại/không là truy vấn DB; chống match nhầm đơn.
- Kiểm tra: tập field hiển thị nằm trong allowlist (không bung data nhạy cảm ngoài phần cần thiết).
  Vì sao nên giao cho code: so danh sách field render với allowlist là kiểm tra cứng.
- Kiểm tra: `reply_draft` không bao giờ ở trạng thái "đã gửi"; không có hành động tạo đơn/sửa data trong output.
  Vì sao nên giao cho code: ràng buộc ranh giới quyền hạn — phải đảm bảo 100% bằng rule, không để LLM tùy ý.

### 6. Tiêu chí chấm bằng LLM

Liệt kê **đầy đủ** các tiêu chí semantic mà case này cần có và code không chấm tốt.

Không giới hạn số lượng. Hãy coi như đây là bộ tiêu chí bạn thật sự sẽ dùng để chấm case này.

Chỉ giữ những tiêu chí mà cần đọc hiểu hội thoại, tính hữu ích của summary, hoặc mức độ an toàn của gợi ý.

Mỗi ý nên viết theo dạng:

- Tiêu chí: [criterion]
  Vì sao code không bắt tốt:

- Tiêu chí: `summary` có phản ánh đúng nội dung & ý định thật của khách trong hội thoại không (so với golden).
  Vì sao code không bắt tốt: tóm tắt hội thoại tự nhiên cần đọc hiểu ngữ cảnh, đa lượt; code không đánh giá được "tóm tắt này có đúng ý khách".
- Tiêu chí: `summary` có grounded vào `lookup_results`, không bịa hồ sơ/đơn/trạng thái không tồn tại trong data tra được.
  Vì sao code không bắt tốt: phát hiện hallucination cần đối chiếu nghĩa giữa câu summary và dữ liệu nguồn — không phải so chuỗi.
- Tiêu chí: phân biệt đúng intent (vd hỏi tình trạng đơn/hậu mãi vs nhu cầu mua mới) — không nhầm intent dẫn tới gợi ý sai hướng.
  Vì sao code không bắt tốt: nhận diện intent từ ngôn ngữ thực (kể cả tiếng Việt không dấu) cần hiểu nghĩa, không phải keyword.
- Tiêu chí: `suggested_next_step` có hợp lý và an toàn không — đúng lúc gợi ý hỏi thêm khi mơ hồ, không xui upsell/chốt khi data chưa chắc.
  Vì sao code không bắt tốt: "đúng thời điểm / đúng mức độ chắc chắn" là phán đoán ngữ cảnh, vượt khả năng rule cứng.
- Tiêu chí: cách diễn đạt cảnh báo có đủ rõ để nhân viên hiểu mức độ không chắc chắn không (không trấn an giả).
  Vì sao code không bắt tốt: chất lượng/độ rõ của lời cảnh báo là đánh giá ngữ nghĩa, code chỉ kiểm được "có cờ warning hay không".

### 7. Human / Expert Review

- Ai cần review?
- Review những case nào?
- Có cần domain expert không? Nếu không, vì sao?

**Trả lời của bạn:**

Đừng chỉ ghi tên team review. Hãy giải thích vì sao đúng nhóm đó cần xem, và họ đang kiểm tra rủi ro gì.

> **Ai review:** Trưởng nhóm Sales Ops / CRM Ops (người hiểu cấu trúc dữ liệu CRM-OMS và quy trình bán hàng).
>
> **Review những case nào:**
> - Case `multiple_matches` / `data_conflict` / `not_found` — kiểm tra Copilot có cảnh báo đúng thay vì tự chốt (đây là rủi ro match nhầm người, nguy hiểm nhất).
> - Case có `reply_draft` hoặc `suggested_next_step = reply_draft` — soi xem nháp có grounded và an toàn không trước khi tin dùng.
> - Mẫu ngẫu nhiên case `single_match` để dò regression và calibrate LLM judge định kỳ.
>
> **Vì sao đúng nhóm này:** họ biết "đúng hồ sơ/đúng đơn" nghĩa là gì trong dữ liệu thật, biết khi nào trùng SĐT do nhập trùng, và hiểu ranh giới hành động được phép của sales. Rủi ro họ kiểm: match nhầm người/đơn, tóm tắt sai dẫn tới trả lời sai khách, và gợi ý vượt quyền (tự chốt/tự gửi).
>
> **Có cần domain expert không? Không.** Đây là bài toán vận hành ops/sales/CRM, không đụng chuyên môn ngành đặc thù (y khoa, pháp lý). "Đúng/sai" ở đây là kiến thức quy trình nội bộ mà Sales Ops đã có, nên human review vận hành là đủ; thêm expert chuyên sâu không tương xứng chi phí.

Nếu chọn **có domain expert**, bạn phải làm thêm 2 phần dưới đây. Nếu **không cần domain expert**, hãy ghi `Không áp dụng` và giải thích 1 câu.

#### 7A. Màn hình cho Domain Expert (ASCII)

Mock một màn hình review cho expert.

Expert cần thấy tối thiểu:

- AI đã match hoặc gợi ý gì,
- dữ liệu nguồn hoặc bằng chứng nào expert cần nhìn lại,
- expert có thể duyệt / sửa / chặn hành động ở đâu.

**Trả lời của bạn:**

```text
Không áp dụng.
```

Không áp dụng — case này chỉ cần human review từ Sales Ops/CRM Ops; không có quyết định đòi chuyên môn ngành đặc thù nên không lập màn hình/tiêu chí domain expert riêng.

#### 7B. Tiêu chí review của Domain Expert

Liệt kê các tiêu chí domain expert sẽ dùng để duyệt case này.

Không áp dụng (xem giải thích ở phần 7).

### 8. Release Gate

Đề xuất release gate phù hợp cho case này. Nêu rõ điều kiện chặn, ngưỡng chất lượng tối thiểu, và trường hợp cần human review.

**Release Gate đề xuất** (chạy trên reference dataset trước khi ship version prompt/model mới):

*Điều kiện chặn cứng (fail bất kỳ ⇒ block ship):*
- Tỉ lệ pass các **code check** (schema, enum, trích/normalize tín hiệu, allowlist field nhạy cảm) = **100%**.
- **0 case** hiển thị một hồ sơ/đơn như đã chắc chắn khi `match_status ≠ single_match` (chống match nhầm người — invariant an toàn).
- **0 case** output có hành động tự gửi tin / tự tạo đơn / sửa data.

*Ngưỡng chất lượng tối thiểu (LLM judge đã calibrate + golden):*
- **Recall cảnh báo = 100%** trên nhóm case cần cảnh báo (`multiple_matches`, `data_conflict`, `not_found`) — thà thừa cảnh báo còn hơn bỏ sót.
- Độ chính xác trích/normalize tín hiệu ≥ **98%**.
- Tỉ lệ `summary` bị đánh "có hallucination" ≤ **5%**.
- Tỉ lệ `suggested_next_step` bị đánh "không an toàn/sai ranh giới" = **0%** (hoặc chặn ship nếu xuất hiện).

*Trường hợp bắt buộc human review:*
- Mọi case ở production có `warnings` hoặc có `reply_draft` được Sales Ops kiểm trong giai đoạn pilot.
- LLM judge lệch chuẩn (precision/recall vs human < 0.8) ⇒ quay lại human label trước khi tin số.
- Có regression so với version trước trên bất kỳ case ambiguity/mâu thuẫn nào.

### 9. Kế hoạch chạy thử và dự toán chi phí

Làm phần này với giả định team của bạn vừa nhận đề bài Copilot này từ công ty.

Bạn là PM phụ trách đề xuất cách xây bộ eval, cách chạy thử, và chi phí cần xin để trả lời câu hỏi:

- hướng làm này hiện chính xác tới đâu
- có đủ an toàn và hữu ích để đem đi đề xuất tiếp hay chưa
- và với một khoản budget thử nghiệm nhỏ, team có thể chứng minh được gì

README của folder này chỉ cho khung tính. Hãy giữ cách tính đơn giản: phần người tính bằng `giờ công`, phần máy tính bằng `chi phí API key` tính từ **giá thật** của model / dịch vụ bạn chọn.

Để làm phần này, bạn cần tự tính và nêu rõ:

- giá API thật bạn dùng để tính
- tổng số cases pilot dự kiến
- tổng số lần chạy / lặp lại dự kiến
- tổng giờ PM / thiết kế eval
- tổng giờ sales ops / CRM ops
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
- và vì sao plan này đủ để chứng minh Copilot có thể pilot được.

**Kế hoạch chạy thử và dự toán (Case 2 — Sales Copilot):**

*Giả định quy mô:*
- Reference dataset pilot: **90 cases** (5 seed + 8 tình huống SC v0 + 5 edge case + mở rộng các lát cắt: match rõ, ambiguity, thiếu info, mâu thuẫn, action safety).
- Số lần chạy/lặp: **40 lần** ⇒ **90 × 40 = 3.600 lượt**.
- Mỗi lượt: ~**2.000 token input** (tin nhắn + lịch sử + data CRM/OMS giả lập + system prompt) + ~**500 token output** (summary + signals + warnings + next step). LLM judge: ~**1.500 token input** + ~**200 token output**.

*Giá API thật (giá công khai Anthropic, tham chiếu qua skill `claude-api`):*
- **Claude Haiku 4.5** cho Copilot: **$1 / 1M input, $5 / 1M output** (đủ cho trích tín hiệu + tóm tắt; rẻ để chạy nhiều lượt).
- **Claude Sonnet 4.6** cho LLM judge: **$3 / 1M input, $15 / 1M output**.

*Chi phí API:*
- Copilot (Haiku): input 3.600 × 2.000 = 7,2M → $7,2; output 3.600 × 500 = 1,8M → $9,0. ⇒ ~**$16,2**.
- Judge (Sonnet): input 3.600 × 1.500 = 5,4M → $16,2; output 3.600 × 200 = 0,72M → $10,8. ⇒ ~**$27,0**.
- **Tổng API ≈ $43** (làm tròn ~$50 dự phòng prompt dài/lặp thêm).

*Giờ công (người):*
- PM / thiết kế eval: **22 giờ**.
- Sales Ops / CRM Ops (dựng data CRM-OMS giả lập + dán nhãn golden 90 cases + calibrate judge): **24 giờ**.
- Kỹ thuật (regex trích tín hiệu giả lập + connector DB giả + runner + chạy): **28 giờ**.
- Human review (soi case ambiguity/mâu thuẫn/reply_draft): **16 giờ**.
- Domain expert: **0 giờ** (không cần — đã giải thích ở phần 7).
- Tổng giờ công ≈ **90 giờ**.

*Tổng kết:*
- **Chi phí API ≈ $50.** Chi phí chính là **~90 giờ công người** (cao hơn Case 1 vì phải dựng data CRM/OMS giả lập và logic lookup).
- **Thời gian dự kiến: ~2–3 tuần.**

Mình dùng giá API công khai của Anthropic (Haiku 4.5 + Sonnet 4.6). Với ~90 cases × 40 lần chạy, tiền máy chỉ ~$50; budget xin chủ yếu là giờ công (~90 giờ). Plan này đủ để chứng minh Copilot pilot được vì nó đo được hai chỉ số sống còn — **độ chính xác lookup/normalize** và **recall cảnh báo (ambiguity/mâu thuẫn)** — đồng thời kiểm tra ranh giới hành động (không tự gửi/tự chốt) trước khi đề xuất tích hợp CRM thật.

---
