# Đặc tả project: Trích xuất hóa đơn tiếng Việt bằng AI

Bản gốc (có sơ đồ, cập nhật liên tục): https://claude.ai/artifact/QKsv1TePa5nxyvZrGw5bcF
File này là bản sao ngày 2026-11-06. Nếu sửa đặc tả, cập nhật cả hai nơi.

## Tổng quan

Project xây một web app nhận ảnh hóa đơn tiếng Việt và trả về dữ liệu có cấu trúc (JSON, Excel), kèm kiểm tra tự động và số đo độ chính xác. Mục tiêu là một sản phẩm portfolio hoàn chỉnh trong khoảng 5 tuần, dùng để apply vị trí AI Engineer fresher, IT ngân hàng và công ty product.

**Bài toán:** nhập liệu hóa đơn, chứng từ bằng tay tốn thời gian và dễ sai. Ngân hàng, công ty kế toán, app quản lý chi tiêu đều cần tự động hóa việc này.

**Người dùng giả định:** nhân viên kế toán hoặc nghiệp vụ cần số hóa hóa đơn; người dùng cá nhân muốn ghi lại chi tiêu.

**Kỹ năng project chứng minh:**

- Dùng Vision LLM để đọc ảnh và trả về structured output theo schema.
- Không tin AI tuyệt đối: kiểm tra kết quả bằng code (tổng tiền, định dạng ngày).
- Đánh giá có số liệu trên bộ dữ liệu có đáp án.
- Sinh dữ liệu tổng hợp để có nhãn chính xác mà không gán tay.
- Làm việc hiệu quả với AI coding assistant và giải thích được mọi phần code.

## Phạm vi

Chỉ làm một loại chứng từ: hóa đơn bán lẻ (siêu thị, cửa hàng, quán ăn). Mở rộng sang loại khác chỉ khi phần chính đã xong và có số đo.

**Có làm:**

- Tải lên 1 ảnh hóa đơn, nhận kết quả trích xuất.
- Kiểm tra hợp lệ tự động và đánh dấu trường nghi ngờ.
- Cho phép sửa tay kết quả trên giao diện.
- Xuất kết quả ra file Excel hoặc CSV.
- Bộ đánh giá với ít nhất 50 hóa đơn có đáp án đã kiểm tra.
- Script sinh hóa đơn tổng hợp.
- Deploy lên web, có link demo công khai.

**Không làm:**

- Đăng nhập, tài khoản người dùng, database.
- Xử lý nhiều trang hoặc file PDF nhiều trang.
- CCCD hay giấy tờ tùy thân thật (rủi ro dữ liệu cá nhân).
- Tự huấn luyện model OCR từ đầu.
- App mobile.

## Dữ liệu cần trích xuất

Mỗi hóa đơn trả về đúng 8 trường dưới đây. Schema này là "hợp đồng" giữa AI và code: mọi bước sau đều dựa vào nó.

| Trường | Kiểu | Bắt buộc | Ví dụ | Luật kiểm tra |
| --- | --- | --- | --- | --- |
| seller_name | chuỗi | Có | Co.opmart Cống Quỳnh | Không rỗng |
| seller_address | chuỗi | Không | 189C Cống Quỳnh, Q1 | |
| invoice_date | ngày (YYYY-MM-DD) | Có | 2026-10-15 | Không ở tương lai |
| invoice_time | giờ (HH:MM) | Không | 18:42 | Đúng định dạng |
| items | danh sách món | Có | xem bên dưới | Ít nhất 1 món |
| subtotal | số nguyên (VND) | Không | 185000 | ≥ 0 |
| discount | số nguyên (VND) | Không | 10000 | ≥ 0 |
| total | số nguyên (VND) | Có | 175000 | Khớp tổng món − giảm giá, sai lệch ≤ 1.000đ |

Mỗi món trong `items` gồm: `name` (tên món), `quantity` (số lượng), `unit_price` (đơn giá), `line_total` (thành tiền). Luật kiểm tra: `quantity × unit_price ≈ line_total`.

Ví dụ đầu ra:

```json
{
  "seller_name": "Co.opmart Cống Quỳnh",
  "invoice_date": "2026-10-15",
  "items": [
    {"name": "Sữa tươi 1L", "quantity": 2, "unit_price": 35000, "line_total": 70000}
  ],
  "total": 70000
}
```

Tiền luôn lưu dạng số nguyên đơn vị đồng, không có dấu chấm phân cách. Đây là lỗi AI hay mắc nhất với hóa đơn Việt Nam (đọc "70.000" thành 70).

## Kiến trúc và công nghệ

Toàn bộ viết bằng Python, chạy trong một app Streamlit, không cần backend hay database riêng.

Luồng chính:

```
Tải ảnh lên (Streamlit)
  → Extractor (Gemini Vision)
  → Kiểm tra schema (Pydantic) ── JSON sai: thử lại, tối đa 2 lần ──┐
  → Validator (tổng tiền, ngày)                                     │
  → Hiện kết quả (cảnh báo + sửa tay) → Xuất file (Excel/CSV)       │
                                                                    │
Luồng đánh giá (chạy riêng, ngoài app):                             │
  Bộ eval (50 hóa đơn có nhãn) → run_eval.py (dùng cùng pipeline) → Bảng kết quả (% đúng theo trường)
```

| Thành phần | Công nghệ | Chi phí |
| --- | --- | --- |
| Ngôn ngữ | Python 3.11+ | Miễn phí |
| Đọc ảnh, trích xuất | Gemini API (model Flash) | Free tier, kiểm tra hạn mức khi đăng ký |
| Định nghĩa và kiểm tra dữ liệu | Pydantic | Miễn phí |
| Giao diện | Streamlit | Miễn phí |
| Sinh hóa đơn tổng hợp | Faker (locale vi_VN), Pillow | Miễn phí |
| Test | pytest | Miễn phí |
| Deploy | Streamlit Community Cloud | Miễn phí |
| Trợ lý code | Claude Code hoặc Cursor | Theo gói bạn dùng |

## Nguồn dữ liệu

Kết hợp ba nguồn: dữ liệu tổng hợp để phát triển nhanh, MC-OCR để thử trên ảnh thật, và 50 hóa đơn kiểm tra tay làm bộ đánh giá chính thức.

| Nguồn | Số lượng | Dùng để | Lưu ý |
| --- | --- | --- | --- |
| Hóa đơn tổng hợp (tự sinh) | 100–200 tấm | Phát triển, thử prompt, test tự động | Biết chính xác đáp án; thêm mờ, xoay, nhiễu cho giống ảnh chụp |
| [MC-OCR 2021](https://people.cs.umu.se/sonvx/files/MCOCR_Preprint.pdf) | khoảng 2.400 ảnh | Thử trên ảnh thật chụp bằng điện thoại | Xem mục tải dữ liệu trên trang cuộc thi |
| [Viet-Receipt-VQA](https://huggingface.co/datasets/5CD-AI/Viet-Receipt-VQA/blob/main/README.md) | 2.034 ảnh | Nguồn dễ tải từ Hugging Face | Nhãn do Gemini sinh ra, không phải đáp án chuẩn |
| Hóa đơn của bạn và người thân | 20–30 tấm | Bổ sung ảnh thật, đa dạng hơn | Che thông tin cá nhân nếu có |

**Bộ đánh giá chính thức (eval set):** chọn 50 hóa đơn ảnh thật (từ MC-OCR hoặc tự chụp), tự kiểm tra và sửa đáp án bằng tay, lưu thành file JSON. Bộ này **không được dùng để chỉnh prompt**, chỉ dùng để đo. Nếu vừa chỉnh vừa đo trên cùng một bộ, con số sẽ đẹp giả.

**Sinh hóa đơn tổng hợp:** script Python tạo dữ liệu ngẫu nhiên (tên cửa hàng, món, giá), vẽ thành ảnh bằng Pillow hoặc HTML, sau đó thêm hiệu ứng. Đáp án được lưu cùng lúc với ảnh.

## Cấu trúc thư mục

```
vn-receipt-extractor/
├── app.py                 # Giao diện Streamlit
├── src/
│   ├── schema.py          # Định nghĩa Pydantic (Receipt, Item)
│   ├── extractor.py       # Gửi ảnh tới Gemini, nhận JSON
│   ├── validator.py       # Luật kiểm tra: tổng tiền, ngày, ...
│   └── prompts.py         # Các phiên bản prompt (v1, v2, ...)
├── data/
│   ├── synthetic/         # Ảnh + đáp án tự sinh
│   ├── eval/              # 50 hóa đơn đánh giá + labels.json
│   └── samples/           # Vài ảnh mẫu cho demo
├── scripts/
│   ├── generate_synthetic.py
│   └── run_eval.py        # Chạy đánh giá, in bảng kết quả
├── tests/
│   └── test_validator.py  # Test cho luật kiểm tra
├── docs/
│   └── decisions.md       # Nhật ký quyết định, lỗi đã gặp
├── CLAUDE.md              # Bối cảnh project cho AI assistant
├── requirements.txt
├── .env.example           # GEMINI_API_KEY=... (không commit .env thật)
└── README.md
```

## Lộ trình

Năm giai đoạn, mỗi giai đoạn khoảng một tuần và kết thúc bằng thứ chạy được.

**Giai đoạn 1 — Trích xuất cơ bản**

- [ ] Tạo repo GitHub, môi trường Python, file `.env` chứa API key
- [ ] Viết `schema.py` theo bảng ở phần Dữ liệu cần trích xuất
- [ ] Viết `extractor.py`: 1 ảnh vào, 1 object `Receipt` ra
- [ ] Chạy thành công trên 5 ảnh mẫu, tự đọc lại và so với ảnh

**Giai đoạn 2 — Kiểm tra hợp lệ và dữ liệu tổng hợp**

- [ ] Viết `validator.py` với các luật trong bảng schema
- [ ] Viết test cho validator, chạy `pytest` qua hết
- [ ] Thêm cơ chế thử lại khi JSON sai định dạng (tối đa 2 lần)
- [ ] Viết `generate_synthetic.py`, sinh được 100 hóa đơn kèm đáp án

**Giai đoạn 3 — Giao diện**

- [ ] Streamlit: tải ảnh lên, hiện ảnh bên trái, kết quả bên phải
- [ ] Trường không qua kiểm tra được tô màu cảnh báo
- [ ] Sửa tay được kết quả, xuất ra Excel/CSV

**Giai đoạn 4 — Đánh giá**

- [ ] Chọn và gán nhãn tay 50 hóa đơn cho bộ eval
- [ ] Viết `run_eval.py`, in bảng độ chính xác theo từng trường
- [ ] So sánh ít nhất 2 phiên bản prompt (hoặc 2 model), ghi kết quả
- [ ] Ghi 3 loại lỗi phổ biến nhất và cách đã xử lý vào `decisions.md`

**Giai đoạn 5 — Hoàn thiện và công bố**

- [ ] Deploy lên Streamlit Community Cloud, có link công khai
- [ ] Viết README theo phần Sản phẩm cuối
- [ ] Quay video demo 1–2 phút
- [ ] Cập nhật CV và LinkedIn

## Đánh giá chất lượng

| Chỉ số | Cách đo | Mục tiêu tham khảo |
| --- | --- | --- |
| Độ chính xác theo trường | Số hóa đơn đúng trường đó / tổng số hóa đơn. Tên cửa hàng so khớp gần đúng; ngày và tiền phải đúng tuyệt đối | Tổng tiền ≥ 90% |
| Độ chính xác danh sách món | Số món khớp (tên gần đúng + thành tiền đúng) / tổng số món trong đáp án | ≥ 80% |
| Tỷ lệ phát hiện lỗi | Trong các hóa đơn AI đọc sai, bao nhiêu % bị validator đánh dấu | Càng cao càng tốt |
| Chi phí và thời gian | Tiền API và số giây trung bình mỗi hóa đơn | Ghi lại để so sánh |

Mục tiêu chỉ để định hướng. Con số thật thấp hơn cũng không sao, miễn giải thích được vì sao và đã thử cải thiện ra sao.

## Làm việc với AI

1. Mở đầu mỗi phiên bằng việc cho AI đọc `CLAUDE.md` (Claude Code tự đọc file này).
2. Mỗi prompt một việc, nêu rõ file nào, đầu vào, đầu ra.
3. Yêu cầu AI hỏi lại nếu có gì chưa rõ, trước khi viết code.
4. Chạy code ngay sau khi AI viết. Lỗi thì dán toàn bộ thông báo lỗi.
5. Hỏi "giải thích đoạn này" cho mọi phần chưa hiểu. Không commit code mình không giải thích được.
6. Commit sau mỗi bước chạy được, message ngắn, rõ nghĩa.
7. Khi AI sai mà mình phát hiện, ghi vào `decisions.md`.

**Prompt mẫu:**

Giai đoạn 1:
> Đọc CLAUDE.md. Viết `src/schema.py` định nghĩa Pydantic model `Item` và `Receipt` theo bảng schema. Tiền là số nguyên VND. Sau đó viết `src/extractor.py` có hàm `extract_receipt(image_path) -> Receipt`, gửi ảnh tới Gemini với structured output. Giải thích ngắn từng phần sau khi viết.

Giai đoạn 2:
> Viết `src/validator.py` có hàm `validate(receipt) -> list[str]` trả về danh sách cảnh báo theo các luật trong CLAUDE.md. Viết thêm test trong `tests/test_validator.py`, gồm cả trường hợp hợp lệ và không hợp lệ. Chưa sửa file khác.

Giai đoạn 3:
> Viết `app.py` bằng Streamlit: tải ảnh lên, gọi `extract_receipt` và `validate`, hiện ảnh bên trái và bảng kết quả có thể sửa bên phải, trường có cảnh báo tô màu vàng, có nút tải Excel. Giữ code đơn giản, không thêm tính năng ngoài yêu cầu.

Giai đoạn 4:
> Viết `scripts/run_eval.py`: đọc `data/eval/labels.json`, chạy extractor trên từng ảnh, tính độ chính xác theo từng trường như mô tả trong CLAUDE.md, in bảng kết quả và lưu ra `results/<ngày>_<phiên bản prompt>.json`. Trước khi viết, hãy hỏi tôi cách so khớp tên cửa hàng.

Khi gặp lỗi:
> Tôi chạy `python app.py` và nhận lỗi bên dưới. Hãy giải thích nguyên nhân trước, rồi mới đề xuất cách sửa.
> [dán toàn bộ lỗi]

## Sản phẩm cuối và chuẩn bị phỏng vấn

**README cần có:** một câu mô tả và link demo ở đầu; GIF hoặc video; sơ đồ kiến trúc; bảng kết quả đánh giá có so sánh phiên bản; mục "Quyết định thiết kế"; mục "Hạn chế và hướng phát triển"; hướng dẫn chạy trong 3–4 lệnh.

**Dòng CV mẫu** (điền số thật sau giai đoạn 4):
> Xây dựng hệ thống trích xuất hóa đơn tiếng Việt bằng Vision LLM và structured output (Pydantic), có lớp kiểm tra tự động phát hiện X% lỗi của model; đánh giá trên 50 hóa đơn thực tế, đạt Y% độ chính xác tổng tiền; sinh 100+ hóa đơn tổng hợp để test; deploy bằng Streamlit.

**Câu hỏi phỏng vấn cần chuẩn bị:**

- Vì sao không để AI tự tính tổng tiền và tự kiểm tra?
- Lỗi phổ biến nhất của model là gì, bạn xử lý thế nào?
- Vì sao không dùng nhãn có sẵn của Viet-Receipt-VQA làm đáp án?
- Nếu ngân hàng cần xử lý 10.000 hóa đơn mỗi ngày thì thay đổi gì? (xử lý theo lô, hàng đợi, cache, model rẻ hơn, người duyệt các ca bị cảnh báo)
- Dữ liệu hóa đơn có thông tin nhạy cảm, bạn bảo vệ thế nào?
- Bạn dùng AI viết code thế nào, và làm sao chắc code đó đúng?
