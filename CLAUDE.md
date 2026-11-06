# CLAUDE.md — Bối cảnh project cho AI assistant

Đọc file này trước khi làm bất kỳ việc gì trong repo. Đặc tả đầy đủ nằm ở `docs/SPEC.md`.

## Project là gì

`vn-receipt-extractor`: web app nhận ảnh hóa đơn bán lẻ tiếng Việt (siêu thị, cửa hàng, quán ăn) và trả về dữ liệu có cấu trúc (JSON, Excel/CSV), kèm kiểm tra hợp lệ tự động và bộ đánh giá độ chính xác.

Đây là project portfolio của một sinh viên IT mới ra trường, dùng để apply vị trí AI Engineer fresher, IT ngân hàng và công ty product. Ưu tiên: **đơn giản, chạy được, đo được, giải thích được**. Không thêm tính năng ngoài phạm vi.

## Người làm project

- Đọc hiểu code ở mức cơ bản, chưa nhiều kinh nghiệm thực tế.
- Sau mỗi đoạn code, giải thích ngắn gọn bằng tiếng Việt: đoạn đó làm gì và vì sao viết như vậy.
- Nếu yêu cầu chưa rõ, **hỏi lại trước khi viết code**.
- Mỗi lần chỉ làm đúng việc được giao, không tự sửa file khác nếu không được yêu cầu.
- Ưu tiên code dễ đọc hơn code ngắn gọn hay "thông minh".

## Công nghệ

| Thành phần | Dùng |
| --- | --- |
| Ngôn ngữ | Python 3.11+ |
| Đọc ảnh, trích xuất | Gemini API qua SDK `google-genai` (`from google import genai`), dùng structured output với Pydantic schema. **Không** dùng SDK cũ `google-generativeai`. |
| Model | Đọc từ biến môi trường `GEMINI_MODEL` (mặc định một model Flash). Không hard-code tên model. |
| Kiểm tra dữ liệu | Pydantic v2 |
| Giao diện | Streamlit |
| Sinh dữ liệu tổng hợp | Faker (locale `vi_VN`), Pillow |
| Xuất file | pandas + openpyxl |
| Test | pytest |
| Deploy | Streamlit Community Cloud |

API key đọc từ file `.env` (biến `GEMINI_API_KEY`) bằng `python-dotenv`. Không bao giờ in key ra log, không commit `.env`.

## Cấu trúc thư mục

```
app.py                     # Giao diện Streamlit
src/schema.py              # Pydantic models: Item, Receipt
src/extractor.py           # extract_receipt(image_path) -> Receipt
src/validator.py           # validate(receipt) -> list[str] (danh sách cảnh báo)
src/prompts.py             # Các phiên bản prompt: PROMPT_V1, PROMPT_V2, ...
data/samples/              # Vài ảnh mẫu cho demo
data/synthetic/            # Ảnh + đáp án tự sinh
data/eval/                 # 50 hóa đơn đánh giá + labels.json
scripts/generate_synthetic.py
scripts/run_eval.py        # Chạy đánh giá, in bảng kết quả, lưu vào results/
tests/test_validator.py
docs/SPEC.md               # Đặc tả đầy đủ
docs/decisions.md          # Nhật ký quyết định và lỗi đã gặp
```

## Schema đầu ra (hợp đồng giữa AI và code)

`Receipt`:

| Trường | Kiểu | Bắt buộc | Luật kiểm tra |
| --- | --- | --- | --- |
| seller_name | str | Có | Không rỗng |
| seller_address | str \| None | Không | |
| invoice_date | date (YYYY-MM-DD) | Có | Không ở tương lai |
| invoice_time | str \| None (HH:MM) | Không | Đúng định dạng HH:MM |
| items | list[Item] | Có | Ít nhất 1 món |
| subtotal | int \| None (VND) | Không | ≥ 0 |
| discount | int \| None (VND) | Không | ≥ 0 |
| total | int (VND) | Có | ≈ tổng line_total − discount, sai lệch ≤ 1.000đ |

`Item`: `name: str`, `quantity: float`, `unit_price: int`, `line_total: int`. Luật: `quantity × unit_price ≈ line_total` (sai lệch ≤ 1.000đ).

Quy ước quan trọng:

- Tiền luôn là **số nguyên đơn vị đồng**. Hóa đơn Việt Nam dùng dấu chấm phân cách hàng nghìn ("70.000" = 70000, không phải 70). Prompt phải nói rõ điều này.
- Trường không đọc được thì để `None`, không đoán.
- Validator **không** sửa dữ liệu, chỉ trả về danh sách cảnh báo (chuỗi tiếng Việt dễ hiểu). Giao diện dùng cảnh báo để tô màu trường nghi ngờ.

## Nguyên tắc thiết kế

1. **Không tin AI tuyệt đối.** Mọi phép tính (tổng tiền, kiểm tra ngày) làm bằng code trong `validator.py`, không nhờ LLM.
2. **JSON sai định dạng thì thử lại**, tối đa 2 lần, sau đó báo lỗi rõ ràng.
3. **Bộ eval (`data/eval/`) chỉ dùng để đo**, không dùng để chỉnh prompt. Chỉnh prompt trên dữ liệu tổng hợp hoặc `data/samples/`.
4. **Prompt có phiên bản** trong `src/prompts.py`, để so sánh v1 với v2 bằng `run_eval.py`.
5. Kết quả eval lưu thành `results/<YYYY-MM-DD>_<prompt_version>.json`.

## Đánh giá

- Độ chính xác theo trường: tên cửa hàng so khớp gần đúng (hỏi người dùng ngưỡng trước khi cài đặt); ngày và tiền phải khớp tuyệt đối.
- Độ chính xác danh sách món: tên gần đúng + `line_total` đúng.
- Tỷ lệ phát hiện lỗi: trong các hóa đơn AI đọc sai, bao nhiêu % bị validator cảnh báo.
- Chi phí và thời gian trung bình mỗi hóa đơn.

## Quy ước code

- Code, tên biến, comment trong code: tiếng Anh. Thông báo hiển thị cho người dùng và giải thích trong chat: tiếng Việt.
- Có type hints cho mọi hàm. Hàm ngắn, mỗi hàm một việc.
- Không thêm thư viện mới ngoài `requirements.txt` khi chưa hỏi.
- Viết test cho `validator.py` trước khi coi giai đoạn 2 là xong.

## Không làm

Đăng nhập, database, PDF nhiều trang, CCCD hay giấy tờ tùy thân thật, tự huấn luyện model OCR, app mobile.

## Tiến độ hiện tại

- [ ] Giai đoạn 1 — Trích xuất cơ bản
- [ ] Giai đoạn 2 — Kiểm tra hợp lệ và dữ liệu tổng hợp
- [ ] Giai đoạn 3 — Giao diện
- [ ] Giai đoạn 4 — Đánh giá
- [ ] Giai đoạn 5 — Hoàn thiện và công bố

(Cập nhật mục này khi xong mỗi giai đoạn.)
