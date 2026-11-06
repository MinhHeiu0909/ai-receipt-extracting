# Nhật ký quyết định và lỗi đã gặp

Ghi ngắn mỗi khi: đưa ra một quyết định thiết kế, gặp lỗi đáng nhớ, hoặc phát hiện AI viết sai. Đây là nguồn câu chuyện cho phỏng vấn.

Mẫu:

```
## YYYY-MM-DD — Tiêu đề ngắn
- Bối cảnh: đang làm gì
- Vấn đề / lựa chọn: gặp gì, có những phương án nào
- Quyết định: chọn gì và vì sao
- Kết quả: đo được gì (nếu có)
```

---

## 2026-11-06 — Chọn đề tài và phạm vi

- Bối cảnh: cần một project portfolio cho vị trí AI Engineer fresher, IT ngân hàng, công ty product.
- Lựa chọn: AI Trip Planner hoặc trích xuất hóa đơn tiếng Việt.
- Quyết định: trích xuất hóa đơn. Sát nhu cầu ngân hàng và fintech, đo lường được, ít code hơn. Chỉ làm hóa đơn bán lẻ trước.
- Dữ liệu: kết hợp hóa đơn tổng hợp (tự sinh, có đáp án chính xác), MC-OCR 2021 / Viet-Receipt-VQA (ảnh thật), và 50 hóa đơn kiểm tra tay làm bộ eval. Không dùng nhãn Viet-Receipt-VQA làm đáp án vì nhãn đó do Gemini sinh ra.
