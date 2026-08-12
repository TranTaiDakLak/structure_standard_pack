# Web Backend — Node.js + Express

## Khi nào dùng

- API JS/TS nhẹ, tốc độ dựng nhanh
- Hợp tool nội bộ, admin API, BFF nhỏ-vừa
- structured khi module tăng

## Có gì trong thư mục này

- `simple.md`: dùng khi cần ship nhanh, repo còn gọn
- `structured.md`: dùng khi code bắt đầu phình và cần boundary rõ hơn

## Baseline lâu dài

- Route/controller mỏng; nghiệp vụ nằm trong services/modules, data access nằm trong repositories/platform.
- Khuyến nghị TypeScript cho dự án sống lâu; nếu dùng JS thuần, cần lint/test nghiêm hơn.
- Config mẫu dùng `.env.example`; secret production không nằm trong repo.
- API production nên có healthcheck, request id, error middleware, validation, và migration script nếu có DB.
- Test tối thiểu service/module unit test và integration test cho route quan trọng.
- Stack này sinh HTTP API nên bị ràng buộc **toàn bộ** [`03-standards/API_RESPONSE_CONTRACT.md`](../../../../03-standards/API_RESPONSE_CONTRACT.md) (contract `api-1`): Tier 0 bắt buộc kể cả `simple.md`; Tier 1 và Tier 2 áp theo **điều kiện của dự án** (có consumer ngoài repo, endpoint đụng tiền, collection lớn dần...), không theo mode thư mục — chọn `simple` không miễn Tier 2, xem section 3 của chuẩn.
- Server đặt `retryable` theo **bản chất lỗi** (tạm thời hay không), KHÔNG hàm ý client được phát lại request: `timeout`/`dependency_failed` vẫn `true` dù việc có thể đã chạy xong ở server, nên POST không idempotent chỉ được client tự động phát lại khi có `Idempotency-Key` — section 6.1b + Tier 2 của chuẩn.
- Express mặc định trả HTML cho 404 và error chưa bắt — phải override bằng catch-all 404 + error middleware 4 tham số đăng ký cuối cùng.

## Ghi chú

- Pack này ghi theo JS/Node tổng quát.
- Có thể dùng TS nhưng giữ cùng logic thư mục.
