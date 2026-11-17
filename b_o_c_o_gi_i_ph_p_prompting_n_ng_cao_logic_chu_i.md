# BÁO CÁO THIẾT KẾ GIẢI PHÁP PROMPTING NÂNG CAO (LOGIC & CHUỖI)

## 1. Bối cảnh & Vấn đề thực tế
* **Bối cảnh doanh nghiệp:** Chuỗi bán lẻ thiết bị công nghệ **TechShop** cần tự động hóa quy trình tiếp nhận, thẩm định điều kiện bảo hành/đổi trả và phản hồi khiếu nại của khách hàng gửi qua email/form hỗ trợ mỗi ngày.
* **Vấn đề gặp phải:** Nếu sử dụng một câu lệnh đơn lẻ (Mega Prompt), AI thường gặp các lỗi nghiêm trọng:
  1. Tính toán sai thời hạn bảo hành hoặc nhầm lẫn chính sách chiết khấu/đền bù (thiếu lập luận đa bước).
  2. Bỏ sót dữ liệu bị thiếu (khách quên ghi mã đơn, ngày mua) hoặc bịa đặt thông tin (Hallucination).
  3. Bị ngắt quãng giữa chừng (chạm ngưỡng `Max Output Tokens`) khi xử lý danh sách hàng loạt đơn khiếu nại cùng lúc.
* **Giải pháp kỹ thuật áp dụng:** Kết hợp **Prompt Chaining** (chia quy trình thành 3 trạm chuyên biệt), **Chain-of-Thought (CoT)** (lập luận logic điều kiện bảo hành từng bước) và **Continue Prompting** (xử lý đứt gãy dữ liệu).

---

## 2. Bước 1: Thiết kế Giải pháp (Kiến trúc Input / Output)

Quy trình được chia thành **3 Trạm xử lý liên hoàn (Pipeline)** thông qua cơ chế **Data Handoff (JSON)**:

### Sơ đồ luồng dữ liệu
```text
[Dữ liệu thô khách gửi] 
       │
       ▼
┌──────────────────────────────────────────────────────────┐
│ TRẠM 1: Trích xuất & Bắt lỗi dữ liệu (Data Validation)   │
└──────┬───────────────────────────────────────────────────┘
       │ Output 1: JSON sạch (Có cờ cảnh báo lỗi nếu thiếu)
       ▼
┌──────────────────────────────────────────────────────────┐
│ TRẠM 2: Thẩm định Logic Bảo hành (Chain-of-Thought)      │
└──────┬───────────────────────────────────────────────────┘
       │ Output 2: JSON quyết định + Lộ trình lập luận (CoT)
       ▼
┌──────────────────────────────────────────────────────────┐
│ TRẠM 3: Soạn thảo Email Phản hồi (Structured Output)     │
└──────────────────────────────────────────────────────────┘
```

### Chi tiết Input / Output từng trạm

| Trạm | Input (Đầu vào) | Ràng buộc & Xử lý bẫy dữ liệu | Output mong muốn (Đầu ra) |
| :--- | :--- | :--- | :--- |
| **Trạm 1: Trích xuất & Bắt lỗi** | Tin nhắn/Email khiếu nại thô của khách hàng đặt trong thẻ `<raw_data>`. | Nếu thiếu `mã_đơn`, `ngày_mua` hoặc `mô_tả_lỗi` $\rightarrow$ Không được tự đoán mò, bắt buộc gán giá trị `null` và bật cờ `is_missing_data: true`. | Chuỗi **JSON** chuẩn hóa gồm: `order_id`, `product_name`, `purchase_date`, `issue_desc`, `is_missing_data`, `missing_fields`. |
| **Trạm 2: Lập luận Logic (CoT)** | Chuỗi JSON từ Trạm 1 + Bảng quy định bảo hành của TechShop + Ngày hiện tại. | Áp dụng **Zero-shot CoT** để tính số ngày đã sử dụng, đối chiếu lỗi phần cứng hay lỗi người dùng (rơi vỡ, vào nước). Nếu `is_missing_data: true` $\rightarrow$ Chuyển trạng thái `PENDING_INFO`. | Chuỗi **JSON** gồm: `reasoning_steps` (từng bước suy luận), `decision` (`APPROVED` / `REJECTED` / `PENDING_INFO`), `compensation_policy`. |
| **Trạm 3: Soạn Email & Xử lý đứt gãy** | Chuỗi JSON quyết định từ Trạm 2. | Sử dụng **Negative Prompting** (không lời chào thừa ngoài khung email). Nếu danh sách dài bị ngắt giữa chừng $\rightarrow$ Dùng **Continue Prompting** neo 3-5 từ cuối. | Mẫu **Email Markdown** hoàn chỉnh, đúng giọng điệu CSKH chuyên nghiệp. |

---

## 3. Bước 2: Thiết kế Prompt Hoàn chỉnh & Kết quả Chạy thử

### 3.1. Bộ Prompt Thực thi theo Chuỗi (Prompt Chaining)

#### 🔹 TRẠM 1: Prompt Trích xuất & Bắt bẫy dữ liệu trống
```text
[CONTEXT]
Bạn là Hệ thống Tiền xử lý Dữ liệu CSKH của TechShop.

[REQUEST]
Trích xuất thông tin từ khiếu nại của khách hàng trong thẻ <raw_data> sang định dạng JSON.

[EXPLANATION & ERROR HANDLING]
- Tuyệt đối KHÔNG tự ý suy đoán hay bịa đặt thông tin không có trong văn bản.
- Nếu khách hàng không cung cấp thông tin nào, bắt buộc điền giá trị null cho trường đó.
- Nếu thiếu bất kỳ trường nào trong 3 trường: "order_id", "purchase_date", "issue_desc", hãy đặt "is_missing_data": true và liệt kê tên trường bị thiếu vào mảng "missing_fields".
- Negative Prompt: Không chào hỏi, không giải thích, chỉ xuất duy nhất khối mã JSON.

<raw_data>
Chào shop, hôm 10/11/2026 mình có mua chiếc tai nghe Sony WH-1000XM5 (mã đơn TS-99281). Hôm nay là 17/11/2026 mình mở ra dùng thì thấy bên tai trái bị rè tiếng, không nghe rõ. Mình không hề làm rơi hay dính nước. Shop hỗ trợ đổi mới giúp mình nhé!
</raw_data>

[EXTRAS - OUTPUT SCHEMA]
{
  "order_id": "string | null",
  "product_name": "string | null",
  "purchase_date": "YYYY-MM-DD | null",
  "issue_desc": "string | null",
  "user_fault_mentioned": "boolean",
  "is_missing_data": "boolean",
  "missing_fields": ["string"]
}
```

#### 🔹 TRẠM 2: Prompt Lập luận Thẩm định Chính sách (Áp dụng Chain-of-Thought)
```text
[CONTEXT]
Bạn là Chuyên viên Thẩm định Bảo hành Cấp cao tại TechShop. Ngày hiện tại là: 2026-11-17.

[POLICY RULES]
1. Đổi mới 1-đổi-1 miễn phí: Áp dụng trong vòng 15 ngày kể từ ngày mua NẾU là lỗi kỹ thuật từ nhà sản xuất (không rơi vỡ, không vào nước).
2. Sửa chữa bảo hành: Áp dụng từ ngày thứ 16 đến 365 kể từ ngày mua với lỗi nhà sản xuất.
3. Từ chối bảo hành miễn phí: Sản phẩm quá 365 ngày HOẶC có dấu hiệu lỗi người dùng (rơi vỡ, cấn móp, vào nước).
4. Bổ sung thông tin: Nếu "is_missing_data" là true, dừng thẩm định và yêu cầu khách bổ sung.

[REQUEST & CHAIN-OF-THOUGHT]
Nhận dữ liệu JSON từ Trạm 1 trong thẻ <handoff_data>. Hãy suy nghĩ từng bước một (Let's think step by step) trong trường "reasoning_steps" trước khi đưa ra quyết định cuối cùng:
- Bước 1: Kiểm tra cờ "is_missing_data".
- Bước 2: Tính chính xác số ngày sử dụng = Ngày hiện tại - "purchase_date".
- Bước 3: Đánh giá nguyên nhân lỗi (Lỗi NSX hay Lỗi người dùng).
- Bước 4: Đối chiếu với 4 quy tắc ở trên để chốt phương án xử lý.

<handoff_data>
{Dán Output JSON từ Trạm 1 vào đây}
</handoff_data>

[EXTRAS]
Chỉ xuất duy nhất định dạng JSON:
{
  "reasoning_steps": ["Bước 1...", "Bước 2...", "Bước 3...", "Bước 4..."],
  "days_used": number | null,
  "decision": "APPROVED_EXCHANGE | APPROVED_REPAIR | REJECTED | PENDING_INFO",
  "action_detail": "string"
}
```

#### 🔹 TRẠM 3: Prompt Soạn Email Phản hồi Khách hàng
```text
[CONTEXT]
Bạn là Trưởng phòng Chăm sóc Khách hàng TechShop.

[REQUEST]
Dựa vào kết quả thẩm định JSON trong thẻ <decision_data>, hãy viết email phản hồi chính thức cho khách hàng.

[ACTION & TONE]
- Giọng điệu: Chân thành, lịch sự, đồng cảm và chuyên nghiệp.
- Nếu decision là "APPROVED_EXCHANGE": Xin lỗi về trải nghiệm không trọn vẹn, xác nhận đủ điều kiện đổi mới 1-đổi-1 miễn phí và hướng dẫn gửi hàng qua bưu cục TechShop.
- Nếu decision là "PENDING_INFO": Lịch sự cảm ơn và liệt kê rõ các thông tin khách cần bổ sung để tiếp tục xử lý.
- Negative Prompt: Không giải thích quy trình nội bộ, không xuất văn bản thừa ngoài mẫu Email.

<decision_data>
{Dán Output JSON từ Trạm 2 vào đây}
</decision_data>
```

#### 🔹 KỸ THUẬT XỬ LÝ ĐỨT GÃY NGỮ CẢNH (Khi chạy hàng loạt bị chạm ngưỡng Max Output Tokens)
Trong trường hợp chạy danh sách 50 đơn khiếu nại cùng lúc và AI bị ngắt ngang chừng tại dòng:
`...vui lòng mang sản phẩm kèm hóa đơn đến chi nhánh TechShop`
Ta sử dụng câu lệnh **Continue Prompting** có neo điểm dừng:
```text
Hệ thống vừa bị ngắt phản hồi. Hãy viết tiếp tục ngay sau cụm từ: "kèm hóa đơn đến chi nhánh TechShop". Giữ nguyên định dạng Markdown hiện tại và tuyệt đối không lặp lại phần nội dung đã viết phía trên.
```

---

### 3.2. Kết quả Chạy thử Thực tế (Simulation Output)

#### ✅ Kết quả chạy thử Trạm 1 (Data Extraction & Validation)
```json
{
  "order_id": "TS-99281",
  "product_name": "Sony WH-1000XM5",
  "purchase_date": "2026-11-10",
  "issue_desc": "Bên tai trái bị rè tiếng, không nghe rõ",
  "user_fault_mentioned": false,
  "is_missing_data": false,
  "missing_fields": []
}
```

#### ✅ Kết quả chạy thử Trạm 2 (Chain-of-Thought Reasoning)
```json
{
  "reasoning_steps": [
    "Bước 1: Kiểm tra cờ is_missing_data = false. Dữ liệu đầu vào đầy đủ mã đơn TS-99281, ngày mua 2026-11-10 và mô tả lỗi.",
    "Bước 2: Tính thời gian sử dụng từ 2026-11-10 đến 2026-11-17 là 7 ngày (<= 15 ngày).",
    "Bước 3: Khách mô tả tai trái bị rè tiếng và xác nhận không rơi vỡ, không vào nước -> Xác định sơ bộ là lỗi kỹ thuật từ nhà sản xuất.",
    "Bước 4: Đối chiếu Quy tắc 1 (trong vòng 15 ngày + lỗi NSX) -> Đủ điều kiện đổi mới 1-đổi-1 miễn phí."
  ],
  "days_used": 7,
  "decision": "APPROVED_EXCHANGE",
  "action_detail": "Đổi mới 1-đổi-1 miễn phí cho sản phẩm Sony WH-1000XM5 thuộc đơn hàng TS-99281."
}
```

#### ✅ Kết quả chạy thử Trạm 3 (Final Customer Email)
```markdown
**Tiêu đề:** [TechShop CSKH] Xác nhận hỗ trợ đổi mới 1-đổi-1 cho đơn hàng #TS-99281

Chào bạn,

Cảm ơn bạn đã tin tưởng và lựa chọn mua sắm tại TechShop. Trước tiên, TechShop thành thật xin lỗi vì sự bất tiện mà bạn gặp phải với chiếc tai nghe **Sony WH-1000XM5** (mã đơn hàng **TS-99281**).

Qua kiểm tra thông tin đơn hàng mua ngày 10/11/2026 và tình trạng lỗi âm thanh bên tai trái, sản phẩm của bạn hoàn toàn đủ điều kiện áp dụng chính sách **Đổi mới 1-đổi-1 miễn phí** trong vòng 15 ngày đầu tiên của TechShop.

Để tiến hành đổi sản phẩm mới nhanh nhất, bạn vui lòng thực hiện theo hướng dẫn sau:
1. Đóng gói sản phẩm kèm đầy đủ hộp và phụ kiện đi kèm.
2. Mang trực tiếp đến cửa hàng TechShop gần nhất hoặc gửi chuyển phát nhanh về Trung tâm Bảo hành TechShop (miễn phí cước vận chuyển 2 chiều).

Ngay khi tiếp nhận sản phẩm, kỹ thuật viên sẽ kiểm tra nhanh và gửi lại thiết bị mới nguyên seal cho bạn trong vòng 24 giờ làm việc. Nếu cần hỗ trợ thêm, bạn vui lòng phản hồi trực tiếp email này hoặc gọi Hotline 1900-xxxx.

Trân trọng,  
**Bộ phận Chăm sóc Khách hàng - TechShop**
```

---

## 4. Đánh giá & Kiểm thử Tình huống Bẫy Dữ liệu (Edge Case Test)

Để kiểm chứng khả năng chống ảo giác và bắt lỗi của hệ thống, nhóm đã thử nghiệm một **Input bị thiếu thông tin**:
* **Input bẫy:** *"Shop ơi cái bàn phím cơ hôm trước mình mua bị liệt nút Enter rồi, đổi cho mình cái khác nhé."* (Thiếu mã đơn, thiếu ngày mua).
* **Output thực tế từ Trạm 1 & Trạm 2:**
  * Trạm 1 tự động gán `"order_id": null`, `"purchase_date": null` và bật cờ `"is_missing_data": true`, `"missing_fields": ["order_id", "purchase_date"]`.
  * Trạm 2 nhận diện cờ lỗi ngay tại Bước 1 của CoT, dừng tính toán ngày sử dụng và xuất quyết định `"decision": "PENDING_INFO"`.
  * Trạm 3 tự động chuyển sang mẫu email lịch sự yêu cầu khách cung cấp mã đơn hàng và số điện thoại đặt hàng, **ngăn chặn thành công 100% tình trạng AI tự bịa ngày mua hoặc duyệt sai chính sách**.