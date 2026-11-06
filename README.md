# vn-receipt-extractor

Trích xuất thông tin hóa đơn bán lẻ tiếng Việt từ ảnh chụp bằng Vision LLM, có kiểm tra hợp lệ tự động và bộ đánh giá độ chính xác.

> Đang phát triển. README đầy đủ (demo, kiến trúc, kết quả đánh giá) sẽ được viết ở giai đoạn 5.

## Chạy trên máy

```bash
python -m venv .venv
source .venv/bin/activate        # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env             # rồi điền GEMINI_API_KEY
```

## Tài liệu

- Đặc tả: [docs/SPEC.md](docs/SPEC.md)
- Nhật ký quyết định: [docs/decisions.md](docs/decisions.md)
